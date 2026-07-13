---
layout: default
title: "Building a Codex Plugin, Step by Step"
---

[← Back to index](../index.html)

# Building a Codex Plugin, Step by Step

Codex-style coding agents extend through the same conceptual building blocks covered in [Building a Claude Plugin](building-a-claude-plugin.html) — skills, hooks, and tool connectors — but the packaging and installation flow has its own shape. This article focuses on what's different.

## The three-part bundle

A Codex plugin, as presented to an installer, is typically described as three explicit components:

- **An app** — the connector/consent surface a user authorizes once (an OAuth-style approval step), establishing the plugin's access.
- **An MCP server** — the tool connector that app authorizes access to, giving the assistant a set of callable tools.
- **A set of skills** — the packaged instructions that actually get triggered during a session.

Presenting these as three separate, visible pieces — rather than one opaque bundle — is a deliberate transparency choice. An installer can see exactly what access they're granting (the app), what capability that access unlocks (the MCP server's tools), and what behavior it enables (the skill list) before installing.

## Hooks follow the same shape

Codex's hook mechanism follows the same event-driven pattern as the general model: a config file maps lifecycle event names (session start, after a tool call, etc.) to commands to run, optionally filtered by a matcher. If you've built hooks for one platform, the mental model transfers directly — event name, optional matcher, command to execute, structured feedback on stdout. The concrete config file name and a small number of environment variables differ by platform, so treat those specifics as a lookup, not something to guess by analogy.

## Skill packaging is identical

Nothing about how a skill is packaged changes for Codex versus any other surface — same directory-per-skill convention, same manifest fields (name, description, version, distribution tier), same eval-suite discipline described in [Evaluating Skills](evaluating-skills.html). This is exactly the value of declaring a **surface** as skill metadata (see [Understanding Skills](understanding-skills.html)): the same skill can be marked as valid for multiple surfaces without a surface-specific rewrite, and a loader decides at load time whether a given skill applies to the current surface.

## Installing: source, ref, and scoped paths

Codex's marketplace-add flow asks for slightly more precision than a simple "add this repo" step:

1. **Source** — the repository URL.
2. **Git ref** — which branch (or tag) to track, since a plugin author may want installers pointed at something more stable than the default branch's latest commit.
3. **Sparse paths** — an optional subdirectory filter, so a marketplace can point at just the plugin-relevant portion of a larger repository rather than requiring the entire repo to be structured as a plugin root.

After adding a marketplace this way, available plugins show up in a personal listing, and installing triggers the app's consent/authorization flow before the bundled tools become usable.

## Versioning and delivery are the same discipline

Nothing about semantic versioning or the two-tier (source repo → distribution channel) delivery model described in the Claude article changes for Codex — the version field, the meaning of a prerelease suffix, and the value of a review step between "merged" and "publicly installable" all apply identically. What differs is purely the installer-facing mechanics: which config file names are read, and what the add-marketplace form asks for.

## What to watch for when porting a plugin across surfaces

If you're adapting a plugin originally built for one surface to also work on Codex (or vice versa), the riskiest part isn't the skill content — skill instructions are largely surface-agnostic. It's the **platform-specific plumbing**: hook config file names, any environment variables a hook script relies on, and the exact shape of the marketplace manifest. A mechanical find-and-replace across these files is a reasonable starting point, but verify every environment variable and file path actually resolves on the target platform before considering the port done — a hook silently referencing a variable that only exists on the original platform will fail quietly rather than loudly.

## Mental model

> The building blocks don't change across surfaces — skills, hooks, and tools are the same three ingredients whether you're targeting Claude or Codex. What changes is the installer-facing contract: how explicitly the bundle's components are presented (Codex's three-part app/server/skills view), and the precise mechanics of pointing a marketplace at your content (source, ref, and path scoping). Get the shared ingredients right once, then treat each surface's packaging as a thin, verifiable adapter on top.
