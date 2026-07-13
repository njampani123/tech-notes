---
layout: default
title: "Building a Claude Plugin, Step by Step"
---

[← Back to index](../index.html)

# Building a Claude Plugin, Step by Step

A plugin is how an AI coding/chat assistant is extended in a distributable, installable way. Practically, a plugin bundles three things: [skills](understanding-skills.html) (packaged instructions the assistant loads), **hooks** (scripts that run at specific points in a session), and **tools** (connections to external systems). Here's how each piece fits together and how you actually ship one.

## The three ingredients

| Ingredient | What it is | Where it lives |
|---|---|---|
| **Skills** | Packaged instructions the assistant loads to perform a specific task well | A directory per skill, each with a manifest file |
| **Hooks** | Scripts that run automatically at defined lifecycle events (session start, after a tool call, etc.) | A settings file mapping event names to commands |
| **Tools** | Connections to external systems the assistant can call | A separate connector config, typically pointing at a protocol server |

A plugin doesn't have to include all three — a plugin can be skills-only. Hooks and tool connectors are what let a skill actually *do* something beyond generating text.

## The plugin manifest

At the root of a plugin-authoring repo, a marketplace manifest declares what's available to install. It typically includes:

- top-level metadata — a name, an owning team/contact, a description, and a version (independent of any individual skill's version)
- a repository reference (where the source lives)
- a list of plugin entries, each with its own name, description, a reference to where its content lives, and the skills it bundles

This manifest is what an install command points at — it's the entry point a client reads to know what's installable, not the plugin's actual runtime configuration.

## Packaging skills into the plugin

This uses the same skill-packaging discipline covered in [Understanding Skills](understanding-skills.html): a directory per skill, a manifest file with frontmatter fields like name, description, license, a semantic version, and a distribution tier (public / internal-tooling / private). The plugin manifest just references these skill directories by path — the skill format itself doesn't change because it's shipping as part of a plugin versus standalone.

Good practice carried over from [Evaluating Skills](evaluating-skills.html): every skill bundled in a plugin should have its own eval suite — at minimum, does it *trigger* on the right prompts, and does it *execute* correctly. A plugin shipping untested skills is shipping untested capability to every installer at once.

## Wiring in hooks

Hooks are configured declaratively: a settings file maps **lifecycle event names** to one or more commands to run when that event fires. Two events cover most real use cases:

- **A session-start event** — fires once per session; useful for environment checks (verifying a required dependency is present) and surfacing a warning without blocking the session.
- **A post-tool-use event** — fires after a tool call completes, often filtered by a matcher (e.g., only fire after file-editing tools) — useful for automated review steps, like flagging a missed version bump or running a lint check after an edit.

Hooks communicate back to the assistant through structured output on stdout (a message to surface, or a decision flagged as needing follow-up) — they're a feedback mechanism, not a hard gate that undoes an action already taken, since most hook events fire after the fact.

One subtlety worth knowing before relying on it: hook configuration at different scopes (a shared project-level file vs. a personal override) typically **merges** rather than one replacing the other. To disable a project-level hook for local testing, expect to edit the project's hook config directly — a personal override on top rarely suppresses it.

## Wiring in tools

Tool access is usually a separate concern from the plugin manifest itself: a config file declares named external connectors, each pointing at a server implementing a common protocol so many different clients can talk to it the same way. A skill's instructions then reference tool names in plain language — "call the search tool," "use the read tool" — rather than hardcoding a tool schema into the skill package. This keeps the skill portable: the same instructions work regardless of exactly how the connector is configured in a given environment, as long as a tool by that name is available.

A good habit before finalizing a skill's instructions: actually query the connected tool source to confirm the exact tool and parameter names exist, rather than guessing. Small naming mismatches here are a common source of a skill loading successfully but failing the first time it tries to act.

## Versioning

Give every skill — and the plugin/marketplace manifest itself — a strict semantic version: `MAJOR.MINOR.PATCH`, optionally with a prerelease suffix. The conventional meaning:

- `0.x.x` — experimental, no stability guarantee
- `1.0.0` — first stable release
- `1.x.0` — backward-compatible additions
- `2.0.0` — breaking change to behavior or interface

A prerelease suffix (`-alpha`, `-beta.1`) is a simple way to keep something out of a "stable" distribution channel automatically, without separate manual gating logic — treat the presence of a prerelease tag as a hard signal, not just documentation.

Version bumps are commonly tied to a release action (like pushing a git tag) that triggers the actual publish/sync step — so the version number isn't just informational, it's what downstream automation watches for.

## Delivering through a marketplace

Shipping a plugin to end users is usually a two-tier process, not a single publish step:

1. **Source repo** — where the plugin/skills are authored and validated. Its marketplace manifest is enough for a developer to add it directly for testing, pointing an install command at the repo URL.
2. **Distribution channel(s)** — a separate, filtered copy of the content that real end users actually install. Commonly there are at least two channels: a pre-release/internal one (broader content, faster iteration) and a stable/public one (only content that's passed validation and carries a non-prerelease version). Promoting content from source to a distribution channel is often an automated sync-and-review step, not a direct push — giving a human a chance to review exactly what's about to become publicly installable.

From an end user's perspective, installing generally looks like: add a marketplace source (a git URL), browse what's available, install, and restart the client to activate it. Newly-merged content isn't necessarily instantly visible to installers — there's often a separate indexing/propagation step between "merged" and "discoverable," worth accounting for when timing an announcement.

## Other things worth getting right

- **Keep each skill narrowly scoped.** A skill covering too many workflows in one file becomes both harder for the assistant to trigger correctly and harder to eval meaningfully. Splitting an overloaded skill into two focused ones is almost always the right call.
- **Write specific, assertive descriptions.** A skill's trigger description determines whether the assistant reaches for it at all — vague descriptions get skipped even when the skill would have been the right tool for the job.
- **Validate and eval-test on every release**, not just at initial authoring — a CI gate that runs schema validation and the skill's own eval suite on every change catches regressions before they reach a distribution channel.
- **Treat skill content as supply-chain-sensitive.** Because a plugin's skills run inside every installer's assistant session, avoid anything that fetches and executes remote code unpinned, embeds credentials, or silently sends data to unexpected destinations — the same bar you'd hold any third-party dependency to.
- **Package a standalone artifact for manual testing.** Being able to produce a single distributable file for a plugin/skill (outside the full sync-and-review pipeline) is genuinely useful for testing changes before committing to the automated flow.

## Mental model

> A plugin is a bundle, not a single artifact — skills (what the assistant knows how to do), hooks (what runs automatically at key moments), and tools (what it can reach outside the conversation). Versioning and a two-tier marketplace flow exist to answer one question safely: *is this specific bundle, at this specific version, actually ready for everyone who installs it* — not just for the person who just wrote it.
