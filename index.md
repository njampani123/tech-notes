---
layout: default
title: Tech Notes
---

# Tech Notes

A running collection of notes and articles on Kubernetes, ArgoCD, and platform engineering.

## Articles

- [Kubernetes Controllers, Explained](articles/k8s-controllers.html) — the reconciliation loop pattern, every built-in controller in `kube-controller-manager`, and how custom controllers/Operators extend it.
- [Taints and Tolerations, Explained](articles/taints-and-tolerations.html) — how nodes repel pods, why tolerations alone don't dedicate a node, and the standard label + taint + toleration + affinity pattern.
- [MCP Gateway: Authentication Logic](articles/mcp-gateway-auth.html) — how a Model Context Protocol gateway brokers identity across backend services instead of vaulting credentials.
- [MCP Gateway: Enterprise Integrations](articles/mcp-gateway-integrations.html) — normalizing issue trackers, wikis, and Git hosts behind a shared tool contract.
