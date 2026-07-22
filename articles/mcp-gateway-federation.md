---
layout: default
title: "MCP Gateway: Federating Multiple MCP Servers Behind One Surface"
---

[← Back to index](../index.html)

# MCP Gateway: Federating Multiple MCP Servers Behind One Surface

The [earlier MCP Gateway pieces](mcp-gateway-integrations.html) covered fronting arbitrary backend services — an issue tracker, a wiki, a Git host — behind a gateway that speaks one protocol to the assistant and translates to each backend's native API. This is the closely related but distinct case: what happens when the backends themselves already speak MCP, because every product team in an organization ships its own MCP server. Instead of an adapter translating REST into MCP, the job becomes **federation** — presenting many independently-built MCP servers as one coherent surface, and doing it without letting any of them borrow another's identity.

## Why federate MCP servers specifically

Once more than a couple of product teams each expose their own MCP server, an assistant that wants to use several of them ends up doing the integration work every one of those teams already did once: a separate connection per server, a separate auth handshake per server, and — the part that's easy to underestimate — a separate tool-naming convention per server, since nobody coordinated names in advance. Two backends independently naming a tool `generate` is not a hypothetical; it's the default outcome of independent teams solving similar problems.

A federating gateway's job is to sit in front of N upstream MCP servers and expose one MCP endpoint that a client connects to once. Structurally this looks like a reverse proxy, but the interesting design work is everything a plain reverse proxy wouldn't need: deciding which tools from which upstream are even visible, resolving naming collisions before they become a client-facing bug, and — because the thing being proxied is itself an *authenticated* protocol — making sure identity flows through every hop correctly rather than getting flattened into one shared credential.

## Building a common interface across many upstreams

**Discovery, not hardcoded definitions.** Rather than hand-writing a tool schema for every upstream tool, the gateway asks each upstream what it has (`tools/list`, `resources/list`, `prompts/list`) and republishes what comes back. This means an upstream that adds a parameter or fixes a description doesn't require a matching change in the gateway — the schema is live, not duplicated by hand and inevitably drifting out of sync.

**Deny-by-default visibility.** Nothing from an upstream is exposed unless it's explicitly allowlisted. This flips the usual instinct (expose everything, hide what's sensitive later) and matters because an upstream server is built and evolved by a team that isn't thinking about *this* gateway's audience — a new tool they ship tomorrow shouldn't become visible to every client of the federation without a deliberate decision to include it.

**Renaming and namespacing.** Two independent upstreams naming a tool the same thing is exactly the collision problem above, and the fix is ordinary but has to be enforced automatically: detect the collision at registration time and refuse to silently let one shadow the other, and give each upstream (or specific tool) a rename or a namespacing prefix to resolve it deliberately. This also lets the gateway present a single, consistent naming convention to clients even when the underlying teams never agreed on one.

**Description overrides, independent of the upstream's own docs.** A tool's description is what actually drives whether an assistant reaches for it — and the upstream team's description was written for their own audience, which might not match this gateway's. Being able to override just the description (while the schema and routing stay tied to the real upstream tool) means the gateway can tailor what the assistant sees without asking every upstream team to rewrite their own docs for someone else's audience.

**Schema cleanup at the boundary.** Not everything an upstream returns is useful once it's embedded in a federated tool list — a root-level schema-dialect marker meant for a standalone document, or a redundant top-level description already carried elsewhere in the response, are exactly the kind of metadata worth stripping at the gateway rather than passing through untouched. This is boundary hygiene, not filtering for safety — it just keeps what clients see minimal and non-redundant.

**Progressive exposure via feature flags.** A tool can be published but marked as hidden from the assistant unless a caller explicitly opts in by name. This decouples "the tool exists and is wired up" from "the tool is generally available" — a new capability can sit fully integrated and testable behind a flag, then go generally visible by removing the flag, with no redeploy and no schema change either time.

## A shared front door

Beyond routing calls to the right upstream, it's worth having exactly one **local, non-proxied tool** that every workflow calls first — not routed to any upstream, answered directly by the gateway itself. Its job is to hand back the current operating instructions for the whole federated surface (which conventions apply, how to sequence a multi-tool workflow) and, as a side effect, it's the one call every path passes through regardless of which upstream ends up serving the actual request — making it a natural place to record which capability was invoked, for telemetry, independent of tracking every individual upstream call separately.

This is the same shared-initialization pattern covered in [Understanding Skills](understanding-skills.html) — a single call every session makes before anything else, used to inject cross-cutting context once instead of duplicating it per skill. Federating many upstream MCP servers is exactly the situation where that pattern earns its keep: the gateway is the only participant positioned to know about *all* the upstreams at once, so it's the natural owner of "here's how the whole surface fits together," even though no single upstream could describe that on its own.

## A safer authentication mechanism between hops

Federating MCP servers means there are two distinct trust boundaries, not one, and conflating them is the most common way this design goes wrong:

- **Client → gateway**: the real caller proves who they are once, typically via a standard OAuth2/OIDC bearer token issued by an identity provider.
- **Gateway → each upstream**: a separate hop, per tool call, that should never just mean "forward the exact same token to whichever upstream owns this tool."

Forwarding the client's raw token unchanged to every upstream works, but it means every upstream sees the same long-lived credential regardless of which specific action it's being used for — a broad token doing narrow jobs, N times over. The safer pattern is **on-behalf-of (OBO) token exchange**: the gateway trades the caller's session token for a new, short-lived, narrowly-scoped token — one that's valid only for the specific upstream and scope the current call actually needs — minted just-in-time via a standard token-exchange grant (RFC 8693 is the common basis) and cached briefly, with a safety margin subtracted from its expiry so a cached token is never used right up to the edge of going stale. This is the same broker-not-vault principle from the [gateway authentication article](mcp-gateway-auth.html), applied specifically to the gateway-to-upstream hop rather than just the client-to-gateway one.

**A concrete lesson worth carrying into any implementation of this:** an early version of this kind of gateway shared one HTTP client across concurrent requests as a performance optimization — reusing connections instead of opening a new one per call. It was reverted after a real bug: the shared client's default-headers dictionary, including the `Authorization` header, was mutated in place for every outgoing call. Under real concurrency, two in-flight requests from two different users could race on that mutation — and one user's request could go out carrying *another* user's token. The fix was to make every upstream call use its own short-lived, ephemeral client and session, created fresh and torn down after that one call, never shared across concurrent requests. The performance cost of doing this (opening a connection per call rather than reusing one) turned out to be small relative to the tool call's own latency — a cheap price for guaranteeing that concurrent requests can never cross-contaminate credentials.

**Authorization layered on top of authentication.** Knowing who the caller is isn't the end of it — a tier of caller (say, a trial or limited-access tier versus a fully-entitled one) should see a restricted tool list, not just the full catalogue with a warning label. That restriction has to be enforced in two places, not one: at listing time (so restricted tools are simply invisible) and independently at call time (so a caller who crafts a direct call to a tool they were never shown still gets rejected before the request reaches any upstream at all). Enforcing it in only one of those two places is a gap — a client that can guess or discover a tool's name shouldn't be able to route around a listing-time-only restriction.

## From an embedded proxy to a shared federation layer

It's common for this capability to start life embedded inside one application: that app's own service receives all client traffic and fans out in-process to every sibling backend, doing its own auth, filtering, and routing along the way — simplest thing that works, and perfectly reasonable when there's only one application that needs it. The natural next step, once more than one application wants the same federation behavior, is lifting that responsibility out into a shared gateway layer that many applications sit behind — at which point the original application becomes just one more federated target itself, hosting whatever tools it uniquely owns, rather than the thing doing the federating for everyone else. The tool-filtering, renaming, and OBO-exchange logic described above don't change shape in that move — what changes is *how many callers* get to rely on them without re-implementing any of it themselves.

## Mental model

> Federating MCP servers is a different problem from federating arbitrary APIs behind MCP — every participant already speaks the same protocol, so the gateway's real job is curating a coherent, collision-free surface out of independently-built pieces (discovery, allowlisting, renaming, description overrides) and making sure identity survives the hop into each of them intact. The two hazards that actually bite in practice are the same one wearing two hats: a tool name silently colliding across upstreams, and a credential silently colliding across concurrent requests. Both are solved the same way — refuse to let two things share an identity (a name, a token) that weren't supposed to be the same thing in the first place.
