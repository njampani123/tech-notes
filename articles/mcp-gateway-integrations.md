---
layout: default
title: "MCP Gateway: Enterprise Integrations"
---

[← Back to index](../index.html)

# MCP Gateway: Enterprise Integrations

Once an [MCP gateway](mcp-gateway-auth.html) can authenticate a user and broker credentials, the next problem is integrating each backend service — issue trackers like Jira, wikis like Confluence, and source control like Git hosts — behind a **consistent tool interface**, so the assistant doesn't need bespoke logic per service.

## Why a common interface matters

Without a gateway, an assistant integrating with three services needs three different mental models: Jira's REST API, a wiki's markup and page-tree structure, Git's object model and hosting-provider-specific APIs (PRs, issues, webhooks). A gateway's job is to **normalize these into a small set of tool shapes** — search, read, create, update, comment — so the assistant reasons about "search the wiki" and "search the issue tracker" the same way, even though the underlying calls are completely different.

## Pattern: adapter per backend, shared contract

Each integration is typically its own **adapter**:

- Translates the gateway's normalized tool call (e.g., `search(query, scope)`) into the backend's native API call.
- Translates the backend's native response back into a normalized shape (e.g., a list of `{title, url, snippet}` results) regardless of whether it came from Jira, a wiki, or a Git host.
- Owns backend-specific pagination, rate limiting, and error handling — so a Jira rate-limit response doesn't leak a Jira-specific error type up to the assistant; it becomes a generic "backend temporarily unavailable, retry" signal.

This means adding a fourth integration (say, a second issue tracker) means writing one new adapter against the existing contract — not touching the assistant-facing tool surface at all.

## Issue trackers (e.g., Jira-like systems)

Typical normalized operations:
- **Search** issues by query/filter (project, status, assignee).
- **Read** a single issue's full detail, including comments and linked issues.
- **Create/update** an issue, comment, or transition its status.

The tricky part is usually **field mapping** — issue trackers are highly customizable (custom fields, workflows, per-project schemes), so the adapter needs a translation layer between "the assistant wants to set priority" and whatever field ID that maps to in a given project's configuration. Getting this wrong silently (writing to the wrong field) is worse than failing loudly, so adapters should validate the target schema before writing.

## Wikis / knowledge bases

Typical normalized operations:
- **Search** across pages by keyword or space.
- **Read** a page's content (usually converting rich markup to plain text or markdown for the assistant to reason over).
- **Write/append**, if the integration is read-write — this is the highest-risk operation, since wikis are often a shared source of truth and an incorrect automated edit is easy to miss until much later.

A useful design choice: make write operations require a more explicit scope/approval than read operations (see the authentication article's point on per-tool scoping) — reads are low-risk and can be broad; writes should be narrow and auditable.

## Source control (Git hosts)

Typical normalized operations:
- **Search** code or commit history.
- **Read** file contents at a ref.
- **Open a pull/merge request**, **comment on a PR**, **read CI/check status**.

Git integrations tend to need the richest permission model, since actions like opening a PR or pushing a commit are consequential and hard to fully undo (a merged PR, a force-push). This is exactly the class of action worth flagging back to the human user for confirmation rather than letting the assistant execute autonomously — the gateway can enforce this by marking certain tools as "requires explicit user confirmation" at the protocol level, independent of whether the backend *would* allow the action.

## Failure isolation

A gateway integrating N services should ensure one backend's outage doesn't take down the others. Practical implications:
- Each adapter has its own timeout and circuit breaker — a slow Git host shouldn't stall a wiki search.
- Errors are surfaced per-tool, not as a gateway-wide failure — the assistant should still be able to use the tools that *are* healthy.

This all assumes the backend speaks its own native API and the gateway translates it into MCP. When the backends *already* speak MCP — because several teams each ship their own MCP server — the job shifts from translation to curation; see [MCP Gateway: Federating Multiple MCP Servers](mcp-gateway-federation.html) for that variant.

## Mental model

> The gateway's value isn't just "one connection instead of many" — it's **translating each backend's idiosyncratic API into a shared, predictable contract**, so growing the number of integrations doesn't grow the complexity the assistant (or the developer building on top of it) has to reason about.
