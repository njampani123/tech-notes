---
layout: default
title: Tech Notes
---

# Tech Notes

A running collection of notes and articles on Kubernetes, AI assistant infrastructure, and platform engineering.

## Kubernetes & Docker

- [Kubernetes Controllers, Explained](articles/k8s-controllers.html) — the reconciliation loop pattern, every built-in controller in `kube-controller-manager`, and how custom controllers/Operators extend it.
- [Taints and Tolerations, Explained](articles/taints-and-tolerations.html) — how nodes repel pods, why tolerations alone don't dedicate a node, and the standard label + taint + toleration + affinity pattern.

## MCP Gateway

- [MCP Gateway: Authentication Logic](articles/mcp-gateway-auth.html) — how a Model Context Protocol gateway brokers identity across backend services instead of vaulting credentials.
- [MCP Gateway: Enterprise Integrations](articles/mcp-gateway-integrations.html) — normalizing issue trackers, wikis, and Git hosts behind a shared tool contract.

## Slackbot & RAG

- [Lessons from Building a Slackbot for an AI Assistant](articles/building-a-slackbot.html) — ack-fast/work-async architecture, idempotency, threading, and other hard-won lessons.

## AI Plugins, Skills & Evals

- [Understanding Skills: How AI Assistants Discover and Load Capabilities](articles/understanding-skills.html) — 1P vs 3P skills, classification tiers, surface targeting, and shared init-time context.
- [Evaluating Skills: A Methodology for Testing AI-Assistant Skills End-to-End](articles/evaluating-skills.html) — tool-sequence verification, isolated test identities, multi-dimension LLM-judge scoring, surface × model matrix testing, and failure triage at scale.
- [Building a Claude Plugin, Step by Step](articles/building-a-claude-plugin.html) — how skills, hooks, and tools combine into a plugin, versioning, and the two-tier marketplace delivery model.
- [Building a Codex Plugin, Step by Step](articles/building-a-codex-plugin.html) — the same building blocks, Codex's three-part app/server/skills bundle, and what changes when porting a plugin across surfaces.
