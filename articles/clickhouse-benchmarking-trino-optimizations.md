---
layout: default
title: "Moving Developer-Productivity Analytics from Postgres to ClickHouse (Part 2: Benchmarking, Trino, and Optimization)"
---

[← Back to index](../index.html)

# Moving Developer-Productivity Analytics from Postgres to ClickHouse (Part 2: Benchmarking, Trino, and Optimization)

[Part 1](clickhouse-analytics-architecture.html) covered why we moved Jira analytics off Postgres and how the ClickHouse-on-Kubernetes-with-S3 cluster got built. This part covers how we actually measured it, an unavoidable architectural question we had to answer along the way — whether to query ClickHouse directly or through an existing SQL federation layer — and the two optimizations that had the largest measurable effect on both latency and cost.

## Benchmark methodology

Load was generated with [Locust](https://locust.io/), driving ClickHouse's native HTTP interface directly (`GET /?query=...`, JSON output format, HTTP basic auth) rather than a client library — the simplest possible path to the query engine, and the one any dashboard backend would actually use. Each simulated user waited a random 0.1–2 seconds between requests, and requests were drawn from a weighted mix intended to look like a real productivity-dashboard workload rather than a synthetic benchmark:

| Query type | Weight | What it does |
|---|---|---|
| Simple count | 10 | `count()` over the full dataset |
| Date-range rollup | 8 | Filter by date range, group by month, compute resolved count and average resolution time |
| Multi-column aggregation | 6 | Group by project and status |
| Complex analytical (CTE) | 4 | Multi-stage CTE computing resolution rate and bug percentage |
| Filter / search | 3 | `WHERE` on issue type and status, ordered by creation date, limited |
| Resolution-time percentiles | 2 | Quantile analysis over resolution time |
| Top-assignee analysis | 2 | Group by assignee, filter and order by resolution count |
| System / monitoring | 1 | Lightweight health-check style queries |

Tests ran at 1, 10, 50, 100, and 200 concurrent users, for 3–15 minutes depending on the scenario, against the full 7.35-million-row dataset from Part 1. That range matters: a single-user run establishes a best-case latency floor, but only the higher-concurrency runs reveal where a configuration actually falls over.

## Why Trino entered the picture

The dashboards consuming this data didn't talk to ClickHouse (or Postgres) directly — they went through [Trino](https://trino.io/), a distributed SQL query engine that sits in front of multiple data sources as **catalogs** and lets a client query any of them (or join across them) through one SQL interface. That's the actual reason Trino was already in the picture: it's the federation layer that lets the same application query a Postgres catalog and a ClickHouse catalog side by side, without hardcoding a different client and dialect per backend.

That convenience isn't free, though, and it raised the question this section actually needed to answer: **if the application already goes through Trino for everything else, what does that habit cost when the destination is ClickHouse specifically** — a database whose main selling point is raw query speed? Federation adds a hop and a translation layer between the client and the engine doing the work; the question was whether that hop's cost was negligible, tolerable, or big enough to justify carving out a direct-access path for the latency-sensitive parts of the dashboard.

## Measuring the cost of the federation layer

Running the identical query set both directly against ClickHouse and through Trino's ClickHouse catalog gave a clean answer:

| Metric | Direct ClickHouse | ClickHouse via Trino | Overhead |
|---|---|---|---|
| Average | 266 ms | 2,384 ms | **9.0x** |
| Median (P50) | 230 ms | 2,500 ms | **10.9x** |
| P95 | 480 ms | 5,600 ms | **11.7x** |
| P99 | 1,200 ms | 11,000 ms | **9.2x** |
| Max | 1,319 ms | 14,013 ms | **10.6x** |

The overhead wasn't evenly spread across query types, either — it scaled with query complexity. A trivial `count()` was only ~2.6x slower through Trino; the multi-column aggregation was ~11.5x slower. That pattern points at the actual mechanism: this isn't a fixed per-request tax, it's overhead that compounds with how much query-planning and result-shaping work Trino has to do on top of what ClickHouse already did natively.

**Where the time actually goes:**

- **An extra network hop** — client → Trino coordinator → ClickHouse, instead of client → ClickHouse directly.
- **Query translation** — Trino parses and re-plans the incoming SQL against its own execution model before it ever reaches the ClickHouse connector.
- **Type conversion overhead**, and a real correctness bug alongside it: the ClickHouse connector maps `String` columns to Trino's `varbinary` instead of `varchar`, which means string equality filters (`WHERE issuetype = 'Bug'`) fail outright through Trino. Every query using a string filter or selecting a string column had to be rewritten to avoid it — a functionality loss on top of the latency cost.
- **Connection and result-set marshaling** — ClickHouse's native result format has to be converted into Trino's row format before it reaches the client.

Adding up rough estimates for each of those individually only accounts for roughly a third of the actual measured gap (a few hundred milliseconds of estimated overhead vs. ~2 full seconds observed) — a reminder that a component-by-component explanation of overhead is a useful mental model, not a full accounting; something in the combination costs more than the sum of the named parts.

**One more data point worth including precisely because it looks backwards at first:** we also ran the same style of comparison for Postgres through Trino, and *Postgres via Trino was faster than ClickHouse via Trino* — 441ms average vs. 2,384ms (5.4x), and 520ms vs. 5,600ms at P95 (10.8x). That is not evidence that Postgres beats ClickHouse — every direct-access test in this project shows the opposite by a wide margin. It's evidence that **Trino's Postgres connector is more mature than its ClickHouse connector**: no type-conversion bug, no query simplification tax, a cleaner path end to end. The lesson generalizes past this project — benchmarking "database A vs. database B" through a shared middleware layer actually benchmarks "connector A vs. connector B" as much as it benchmarks the databases themselves, and it's worth stating which one you mean before quoting a number.

## Optimizing ClickHouse: caching on top of S3

Part 1 flagged that object storage trades away local-SSD latency for effectively unlimited, cheap capacity. The single change that clawed back most of that latency — and then some — was enabling ClickHouse's caching layers, which had shipped disabled.

The problem this uncovered first: with caching off, a single query was issuing **412 S3 requests on average** — even though roughly 90% of the dashboard's queries were identical or near-identical repeats of the same handful of shapes in the table above. Every one of those repeats was re-fetching the same objects from S3 from scratch.

Turning on three cache layers — a 10 GB mark cache (index lookups), a 20 GB uncompressed cache (decompressed column data), and a 10 GB query cache (whole-result caching, 1-hour TTL) — cost 40 GB of memory per pod (160 GB across the 4-pod cluster) and changed the picture at 200 concurrent users, sustained for 15 minutes:

| Metric | Without cache | With cache | Improvement |
|---|---|---|---|
| Average response | 1,861 ms | 254 ms | **7.3x** |
| Median response | 1,600 ms | 67 ms | **23.9x** |
| P95 | 4,200 ms | 1,300 ms | **3.2x** |
| Throughput | 68.76 req/s | 168.67 req/s | **2.5x** |
| S3 requests per query | 412.2 | 55.9 | **7.4x fewer** |
| Failures (of ~60–150K requests) | 18 (0.03%) | 0 | eliminated |

The **cost** side of that is what made it an easy call to ship, not just a performance win: fewer S3 requests directly means a smaller S3 bill. Extrapolating the 15-minute test's request volume out to a month put S3 request costs at roughly $29,352/month without caching and $18,511/month with it — about $10,800/month, or ~$130K/year, from a config change with no code or schema changes involved.

Why caching pays off this well specifically here, and might not elsewhere: the workload is read-heavy with a small, repetitive query shape set (a monthly rollup, a project/status breakdown, a handful of dashboard filters) hitting a dataset that's large in aggregate but modest enough per-partition to actually fit in a few tens of gigabytes of cache. A workload dominated by unique, one-off queries wouldn't see anywhere near this benefit — the S3-requests-per-query number wouldn't have started at 412 in the first place if 90% of queries weren't near-duplicates of each other.

## Optimizing the Trino path: the spooling protocol

Trino's older result-delivery model streams every row of a result set back through the coordinator node, which becomes a bottleneck under concurrency — more simultaneous queries means more result data all funneling through one process. A newer **spooling protocol** changes that: results are written out to external storage instead, and clients fetch them asynchronously, freeing the coordinator to spend its time on query planning rather than result shuttling.

Turned on at 100 concurrent users, the throughput and tail-latency numbers moved a lot:

| Metric | Before (no spooling) | After (spooling) | Change |
|---|---|---|---|
| Throughput @ 100 users | 1.25 req/s | 5.62 req/s | **+349%** |
| P99 latency @ 100 users | 177 s | 86 s | **-51%** |
| Median @ 100 users | 33 s | 7.8 s | **-76%** |
| Scaling behavior | throughput *dropped* as users increased | throughput *rose* as users increased | qualitatively fixed |

That last row is the most important one: without spooling, going from 10 users to 100 users made throughput **worse** (2.14 → 1.25 req/s) — a coordinator-bound system getting more congested under load, not less. With spooling, it scaled the right direction.

Here's the part worth stating plainly rather than quietly dropping: **the spooling test and the pre-spooling baseline weren't run under identical conditions.** The baseline ran before ClickHouse's caching layer was enabled; the spooling test ran after. So this result is genuinely the combined effect of spooling *and* caching together — not spooling in isolation — and we don't have a clean, apples-to-apples number that isolates spooling's own contribution. We caught this after the fact, documented it as a known confound rather than re-labeling the number as something it isn't, and left "re-run with cache held constant across both arms" as follow-up work rather than a blocker to shipping. The combined result is still real and still actionable (spooling + caching together made Trino viable at higher concurrency), it's just not the single-variable number a rigorous before/after table implies at a glance — worth remembering any time a "before" and "after" snapshot were captured on different days, because production systems rarely hold still in between.

## Recommendations

Putting the direct-vs-Trino numbers and the two optimizations together, the guidance we settled on:

- **Use direct ClickHouse access for anything latency-sensitive** — an interactive dashboard, a user-facing report — where the 9–11x Trino overhead is unacceptable and there's no need to join against another catalog.
- **Reserve Trino for genuinely federated queries** — joining the ClickHouse analytics store against another catalog in one statement — or for background/batch workloads where a few extra seconds of latency doesn't matter.
- **Enable ClickHouse's caching layers by default** on any S3-backed deployment with a repetitive query shape — the cost is memory, already available with per-pod headroom, and the payoff (latency, throughput, and S3 bill) was too large to leave off.
- **If Trino is in the path, turn on spooling for concurrent workloads** — but measure its standalone effect properly (same cache state on both sides of the comparison) rather than trusting a confounded before/after number, ours included.

## Mental model

> A federation layer and a fast query engine solve different problems, and paying for one doesn't mean you get the benefits of the other for free — measuring the gap between "query the engine directly" and "query it through the layer everything else already goes through" is how you find out whether that convenience is cheap or expensive for a given workload. And two of the biggest wins in this whole exercise — enabling ClickHouse's cache and enabling Trino's spooling protocol — were configuration changes, not code changes: the fastest performance work available was often turning on a feature that already existed, not writing something new.
