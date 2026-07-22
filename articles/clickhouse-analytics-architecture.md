---
layout: default
title: "Moving Developer-Productivity Analytics from Postgres to ClickHouse (Part 1: Motivation & Architecture)"
---

[← Back to index](../index.html)

# Moving Developer-Productivity Analytics from Postgres to ClickHouse (Part 1: Motivation & Architecture)

This is Part 1 of a two-part write-up. This part covers *why* we needed a new analytics store at all and how we deployed and migrated into it. Part 2 covers the benchmarking — what actually got faster, and by how much.

## The problem: measuring developer productivity means querying years of ticket history

A common way an engineering organization tries to understand developer productivity is by mining Jira — cycle time, resolution rate, throughput by team or project, trends over quarters or years. The catch is *where that query runs*. Jira is a live, transactional system serving real-time engineering work — creating tickets, updating status, running workflows. It was never built to also serve as an analytical warehouse.

Once a productivity dashboard needs to compute rollups across years of history, for every team, refreshed regularly, one of two things happens: either every dashboard load fires expensive aggregation queries straight at Jira, competing with the traffic Jira actually exists to serve — or someone builds a separate historical store designed for exactly this kind of read pattern, and stops probing Jira live for it at all. We went with the second option. This isn't a story about Jira being slow; it's the ordinary shape of "operational database" vs. "analytical store" — Jira is optimized for many small transactional reads/writes, and productivity analytics wants the opposite: infrequent writes, but large scans and aggregations over years of rows.

## Why not just keep it in Postgres?

The first cut of this historical store was a straightforward relational database (RDS Postgres) — export the ticket history out of Jira, load it into Postgres, and query that instead of hitting Jira directly. This solved the "don't hammer Jira" problem, but not the underlying performance one: a row-oriented transactional database is still the wrong storage layout for "scan years of rows and aggregate by month, by project, by resolution time." As the historical dataset grew — in our case, several years and multiple millions of ticket records — analytical queries (group-by-month rollups, resolution-time percentiles, cross-project comparisons) got slower in exactly the way row-oriented storage predicts: every query still has to touch full rows to read the handful of columns an aggregation actually needs.

That's the specific gap a **columnar OLAP engine** closes. Reading only the columns a query touches, and being built around scan-and-aggregate access patterns instead of point lookups, is the whole design premise of ClickHouse — so we evaluated it as a drop-in replacement for the *analytics* store, not for Jira itself.

## Why ClickHouse, and why on top of object storage

Two decisions shaped the architecture beyond just "pick a columnar database":

- **Kubernetes**, because that's where the rest of the platform already ran, and a StatefulSet-based deployment gives predictable pod identity for a sharded, replicated database — each shard/replica pairing needs a stable network identity for replication to work at all.
- **S3 as the storage backend**, rather than local disk (EBS-backed persistent volumes), because historical ticket data grows monotonically and doesn't need to live on the same fast, expensive block storage as active query working sets. ClickHouse supports this natively through a **storage policy** that transparently redirects a table's data files to an object-storage disk instead of local disk — the query engine and SQL surface don't change, only where the bytes physically live.

The trade-off is real and worth stating plainly: object storage has meaningfully higher per-request latency than local SSD. Part 2 gets into exactly how much that mattered in practice, and what closed most of the gap.

## Cluster topology

The deployment is a hand-rolled Helm chart (a StatefulSet per shard, rather than a dedicated ClickHouse operator/CRD) with this shape:

| Component | Topology | Purpose |
|---|---|---|
| ClickHouse data pods | 2 shards × 2 replicas (4 pods) | Holds and serves query data |
| Keeper pods | 3 replicas | Coordinates replication (ClickHouse's built-in ZooKeeper-compatible service) |
| Storage | S3 bucket, one prefix per shard/replica | Durable data storage behind the `storage_policy` |

Each ClickHouse pod's storage configuration declares an S3-backed disk and a policy that points at it:

```xml
<storage_configuration>
  <disks>
    <s3_disk>
      <type>s3</type>
      <endpoint>https://s3.amazonaws.com/&lt;bucket&gt;/&lt;prefix&gt;/</endpoint>
      <access_key_id from_env="AWS_ACCESS_KEY_ID"/>
      <secret_access_key from_env="AWS_SECRET_ACCESS_KEY"/>
      <region>&lt;region&gt;</region>
    </s3_disk>
  </disks>
  <policies>
    <s3_policy>
      <volumes>
        <s3_volume><disk>s3_disk</disk></s3_volume>
      </volumes>
    </s3_policy>
  </policies>
</storage_configuration>
```

Any table created with `SETTINGS storage_policy = 's3_policy'` now writes its data parts to that S3 prefix instead of the pod's local volume. ClickHouse organizes those parts as **content-addressable** objects (hashed part names, not human-readable paths) — which is convenient for deduplication and integrity checking, but means you can't just browse the bucket to sanity-check what's there; you query `system.parts` (`disk_name`, `path`, `bytes_on_disk`) from ClickHouse itself instead.

Getting the cluster topology right also meant putting a **network policy** in place restricting pod egress to DNS, HTTPS, and the specific S3 endpoint IP ranges the storage disk needs to reach — object storage over the public internet is a new egress path a purely EBS-backed deployment never needed to think about.

## Migrating the schema and the data

The source data was a set of monthly CSV exports out of Jira's backing database — tens of files, one per month, several years of history, several million ticket rows in total. Two things made the migration itself straightforward once the cluster existed:

**Schema mapping.** Source columns came through as loosely-typed export strings; the ClickHouse schema tightened them into proper types and added a few computed columns as `MATERIALIZED` expressions, so downstream queries never have to repeat this logic:

```sql
CREATE TABLE analytics.issues_local (
    issue_id     UInt64,
    project_id   UInt64,
    reporter     String,
    assignee     String,
    issue_type   String,
    resolution   String,
    status       String,
    created      DateTime,
    resolved_at  Nullable(DateTime),

    -- computed once, at insert time — never recomputed per query
    year_month       UInt32 MATERIALIZED toYYYYMM(created),
    resolution_days  Nullable(Int32) MATERIALIZED
        if(resolved_at IS NULL, NULL, dateDiff('day', created, resolved_at)),
    is_resolved      UInt8 MATERIALIZED if(resolved_at IS NULL, 0, 1)
)
ENGINE = MergeTree()
ORDER BY (project_id, created)
PARTITION BY toYYYYMM(created)
SETTINGS storage_policy = 's3_policy'
```

`PARTITION BY toYYYYMM(created)` turns "years of history" into one partition per month — a query filtered to a date range only has to touch the partitions that overlap it, instead of scanning everything. `ORDER BY (project_id, created)` picks the physical sort order that makes the most common access pattern (per-project, time-ordered) cheap.

**Sharding by month parity.** Rather than a purely random shard assignment, each monthly file was routed to a shard based on whether its month number was even or odd — a simple, deterministic rule that spread roughly half the rows onto each shard without needing a coordination step at load time. Loading itself ran two files at a time in parallel (one per shard), which roughly halved total load time over loading sequentially.

**A first pass at cross-shard querying.** With data split across two shards, queries need a single logical table to hit. The textbook answer is ClickHouse's `Distributed` table engine, which fans a query out to every shard and merges the results transparently. Our first attempt at wiring that up hit an authentication error between the distributed engine and the per-shard connections — rather than resolving it in place, we shipped a hand-built `UNION ALL` view over `remote()` calls to each shard as an interim fallback. It works, but it's a workaround, not the intended path, and it's the first thread in a larger replication story — covered properly in the next section.

## What came next

Getting data in place with a working (if imperfect) cross-shard view was enough to start running real queries and measuring them — which is where [Part 2](clickhouse-benchmarking-trino-optimizations.html) picks up: the actual benchmark methodology, why a SQL federation layer (Trino) entered the picture and what it cost in latency, and the two configuration-only changes that had the biggest effect on both latency and cost.

## Mental model

> The decision to move off a live transactional system for analytics isn't really about that system being "slow" — it's about recognizing that "serve real-time writes" and "scan years of history for aggregation" are different jobs with different optimal storage layouts, and forcing one engine to do both means one of the two jobs always loses. Moving the analytical job to a columnar store on object storage — sharded, replicated, and partitioned by the dimension most queries filter on — is what lets the live system go back to just being a live system.
