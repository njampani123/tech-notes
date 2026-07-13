---
layout: default
title: "Understanding Skills: How AI Assistants Discover and Load Capabilities"
---

[← Back to index](../index.html)

# Understanding Skills: How AI Assistants Discover and Load Capabilities

A **skill** is a packaged capability for an AI assistant — instructions, and often a set of tools, bundled together so the assistant can be taught a new task without retraining the underlying model. As assistants gain the ability to load many skills from many sources, a few architectural questions become unavoidable: who wrote this skill, where is it allowed to run, and what does it need to actually function?

## First-party vs. third-party skills

Skills generally fall into two trust categories:

- **First-party (1P):** built and maintained by the platform owner itself. These tend to have deeper integration and broader default trust, since the same organization controls both the assistant and the skill.
- **Third-party (3P):** built by external developers or partners extending the platform. These need narrower, more explicit access — the platform can't assume the same level of trust for code it didn't write.

The shape of a 1P and 3P skill can look nearly identical (same manifest format, same tool-declaration mechanism) — the difference is in the **trust boundary and default permissions**, not the packaging.

## Classification tiers

A useful pattern is to classify every skill into a small number of distribution/trust tiers, typically declared as metadata rather than inferred from code:

| Tier | Who it's for | What it can touch |
|---|---|---|
| **Public** | End customers of the platform | Only capabilities safe to ship broadly; no access to the platform owner's internal systems |
| **Internal-tooling** | The platform owner's own developers, building or extending skills | Broader access needed to build and test new skills, but still not raw internal-system access |
| **Private** | The platform owner's internal staff only | Access to internal-only tools and systems, never shipped externally |

This tiering matters because it answers a question a loader has to resolve *before* a skill ever runs: is this skill even allowed to be exposed in this context at all? Declaring the tier as metadata (rather than, say, inspecting what APIs the skill's code happens to call) lets a registry or loader make that decision cheaply and safely, without executing untrusted code first.

## Surface targeting

Most platforms don't ship one assistant experience — they ship several: a chat UI, an IDE plugin, a CLI, maybe a chat-app integration. A skill built for "generate a slide deck" may make no sense inside an IDE plugin; a skill built for "refactor this function" may make no sense in a chat app. Declaring a **target surface** as metadata lets the loader filter which skills even get offered in a given context, instead of exposing every skill everywhere and hoping the assistant ignores the irrelevant ones.

## Declaring required capabilities

A skill frequently depends on external tools being available — commonly, tools exposed by a connected tool server (an MCP server, in current terminology). Rather than discovering a missing dependency only when the skill actually tries to call a tool mid-task, a **compatibility/capability declaration** lets the skill state upfront what it needs. The loader can then validate availability before ever exposing the skill to the assistant — failing fast and visibly, instead of failing confusingly halfway through a user's request.

## A shared initialization step

A common pattern across skill systems: before any individual skill executes, a shared **initialization step** runs once per session. This is the natural place to inject cross-cutting context that every skill would otherwise have to duplicate — safety guardrails, tool-use conventions, environment details — and it doubles as a clean point to emit usage telemetry (which surface, which skill, which tools were invoked), since it's the one call every skill path passes through regardless of which skill actually runs.

## Mental model

> Skills need the same three questions answered as any pluggable capability system: *who wrote it* (1P/3P), *where can it run* (classification tier + surface), and *what does it need* (capability declaration). Answering these as declared metadata — rather than inferred behavior — is what lets a loader make safe decisions before a skill ever touches a real user's session.
