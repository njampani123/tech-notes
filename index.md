---
layout: default
title: Tech Notes
---

# Tech Notes

A running collection of notes and articles on Kubernetes, AI assistant infrastructure, and platform engineering.

## Kubernetes & Docker

- [Kubernetes Controllers, Explained](articles/k8s-controllers.html) — the reconciliation loop pattern, every built-in controller in `kube-controller-manager`, and how custom controllers/Operators extend it.
- [Taints and Tolerations, Explained](articles/taints-and-tolerations.html) — how nodes repel pods, why tolerations alone don't dedicate a node, and the standard label + taint + toleration + affinity pattern.

## Data Infrastructure

- [Moving Developer-Productivity Analytics from Postgres to ClickHouse (Part 1: Motivation & Architecture)](articles/clickhouse-analytics-architecture.html) — why querying Jira live for developer-productivity metrics doesn't scale, a sharded/replicated ClickHouse-on-Kubernetes cluster backed by S3 storage policies, and the schema/data migration off monthly Jira CSV exports.
- [Moving Developer-Productivity Analytics from Postgres to ClickHouse (Part 2: Benchmarking, Trino, and Optimization)](articles/clickhouse-benchmarking-trino-optimizations.html) — Locust benchmark methodology, why a Trino federation layer entered the picture and what it cost in latency, and the two config-only optimizations (ClickHouse caching, Trino spooling) that moved the needle most.

## MCP Gateway

- [MCP Gateway: Authentication Logic](articles/mcp-gateway-auth.html) — how a Model Context Protocol gateway brokers identity across backend services instead of vaulting credentials.
- [MCP Gateway: Enterprise Integrations](articles/mcp-gateway-integrations.html) — normalizing issue trackers, wikis, and Git hosts behind a shared tool contract.

## Slackbot & RAG

- [Lessons from Building a Slackbot for an AI Assistant](articles/building-a-slackbot.html) — ack-fast/work-async architecture, idempotency, threading, and other hard-won lessons.

## AI Plugins, Skills & Evals

- [Understanding Skills: How AI Assistants Discover and Load Capabilities](articles/understanding-skills.html) — 1P vs 3P skills, classification tiers, surface targeting, and shared init-time context.
- [Evaluating Skills: A Methodology for Testing AI-Assistant Skills End-to-End](articles/evaluating-skills.html) — tool-sequence verification, isolated test identities, multi-dimension LLM-judge scoring, surface × model matrix testing, and failure triage at scale.
- [Operationalizing Skill Evals: CI Pipelines, Shared Fixtures, and Published Reports](articles/operationalizing-skill-evals.html) — running that methodology continuously across 1P/3P surfaces: shared test fixtures, combined tool+judge verdicts, run tracing, durable artifact storage, gh-pages reports, and a CI pipeline that gates skill-metadata changes pre-merge.
- [Building a Claude Plugin, Step by Step](articles/building-a-claude-plugin.html) — how skills, hooks, and tools combine into a plugin, versioning, and the two-tier marketplace delivery model.
- [Building a Codex Plugin, Step by Step](articles/building-a-codex-plugin.html) — the same building blocks, Codex's three-part app/server/skills bundle, and what changes when porting a plugin across surfaces.
