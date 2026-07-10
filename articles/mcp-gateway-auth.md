---
layout: default
title: "MCP Gateway: Authentication Logic"
---

[← Back to index](../index.html)

# MCP Gateway: Authentication Logic

The [Model Context Protocol (MCP)](https://modelcontextprotocol.io) lets an AI assistant call tools exposed by external servers — a Jira server, a Git host, a wiki. An **MCP gateway** sits in front of many such servers and gives the assistant one connection point instead of dozens. That convenience only works if the gateway also solves authentication cleanly, since every backend service has its own identity model.

## The core problem

A gateway aggregates tools from N backend services. Each service:

- has its own auth scheme (OAuth2, API tokens, SSO-issued session cookies, service accounts),
- expects its own scopes/permissions,
- should only ever see the identity of the actual end user, not a shared "gateway" identity — otherwise every action looks the same in that service's audit log regardless of who triggered it.

So the gateway's job is **not** to invent a new auth system — it's to broker between "who is this user, in the assistant's session" and "what credential does backend service X need to act as that user."

## Common pattern: token exchange, not credential storage

The pattern that scales cleanly is **on-behalf-of token exchange**, not storing every backend credential in the gateway:

1. **User authenticates once** to the gateway (typically OIDC/OAuth2 — SSO login, short-lived session token issued to the assistant).
2. **Per-tool-call, the gateway exchanges that session identity for a scoped, short-lived token** for the specific backend being called — using standards like OAuth2 token exchange (RFC 8693) or a backend-specific service-to-service auth flow.
3. **The backend service sees the real user's identity** (or a clearly-attributed delegated identity), not a generic gateway account — so its own permission model and audit trail stay meaningful.
4. **Tokens are minted per-request or per-session and expire quickly.** The gateway is not a long-term credential vault; it's a broker that trades one proof of identity for another, on demand.

This avoids the two failure modes that plague naive designs:
- **Shared service account for everything** → backend can't tell which user did what, permissions become all-or-nothing.
- **Gateway stores long-lived per-user secrets for every backend** → gateway becomes a high-value target; a breach there compromises every connected system at once.

## Scoping matters as much as authentication

Knowing *who* the user is isn't enough — the gateway also needs to enforce *what they're allowed to do*, per tool:

- A user might be allowed to **read** from a wiki but not **write**.
- A tool that can open Git pull requests should require a stronger scope than one that only reads file contents.
- Scopes should map to the backend's *own* permission model where possible, so "can this user do X" is answered by the source system's rules, not duplicated (and potentially drifted) logic inside the gateway.

The safest default is deny-by-default: a tool is only exposed to a session if that session's identity has been explicitly granted the scope it needs, checked at call time — not just at login time — since permissions can change mid-session.

## Session and token lifecycle

Because an AI assistant session can be long-lived (a conversation spanning many tool calls over minutes or hours), the gateway typically separates two lifetimes:

- **Session identity** — established once at login, revocable centrally (e.g., if the user's SSO session ends, every subsequent tool call should fail closed).
- **Per-backend tokens** — short-lived, re-minted as needed, never outliving the session identity that authorized them.

This means revoking access is a single action (kill the session), not N actions across every backend the session ever touched.

## Mental model

> An MCP gateway's auth layer is a **broker**, not a **vault**. It should hold as little standing credential material as possible, and every backend call should carry a fresh, narrowly-scoped, attributable proof of identity — traded just-in-time from the user's own authenticated session.
