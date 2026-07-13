---
layout: default
title: "Lessons from Building a Slackbot for an AI Assistant"
---

[← Back to index](../index.html)

# Lessons from Building a Slackbot for an AI Assistant

Slack is often the first surface teams put an internal AI assistant behind — it's ubiquitous, event-driven, and low-friction compared to standing up a new UI. But Slack's request model imposes constraints that shape the whole architecture, and most of the hard lessons show up only once real users start hammering on it.

## The core constraint: Slack expects a fast ack

Slack requires your endpoint to acknowledge an event within **3 seconds**, or it treats the request as failed and retries it. An LLM-backed response almost never completes that fast. The resulting pattern is universal:

1. **Ack immediately** — return `200 OK` the instant the event is received, before any real work happens.
2. **Post a placeholder message** ("thinking…" / a loading indicator) so the user has feedback.
3. **Do the actual work asynchronously**, then **edit the placeholder message in place** (`chat.update`) once the real response is ready, rather than posting a new message.

Skipping this and doing the work synchronously in the request handler is the single most common mistake — it works fine in testing and then times out under any real load or slow downstream call.

## Idempotency matters more than expected

Because Slack retries on any timeout or ambiguous failure, your handler **will** receive duplicate events for the same user action. If you don't dedupe by event ID, you'll see the bot occasionally responding twice to one message — confusing and hard to reproduce, because it only shows up when the backend happens to be slow. Track processed event IDs (even a short-TTL cache is enough) and skip repeats.

## Threading is a UX decision, not an implementation detail

Users expect a bot's reply to land in the **same thread** as the message that triggered it, not as a new top-level message in the channel. This means tracking `thread_ts` from the triggering event and always replying into it — including for follow-up turns in a multi-message conversation. Get this wrong and every reply looks like an unrelated new message, and multi-turn context becomes visually incoherent.

## Message size limits force output chunking

Slack messages have a length limit, and rich formatting (Block Kit) has its own layout constraints. An LLM can easily generate a response longer than what fits in one message. Plan for this from the start — either paginate long output across multiple messages, or summarize with a "show more" pattern — rather than truncating unexpectedly at the limit boundary.

## Errors should usually be ephemeral, not public

When something fails downstream (a tool call errors, a backend timeout), posting the raw error into the channel is noisy and looks broken to every channel member. Use **ephemeral messages** (visible only to the requesting user) for error/debug output, and keep the public thread clean — reserve public messages for successful, complete responses.

## Rate limits are per-workspace, not per-request

Slack's API rate limits apply at the workspace level across all your bot's activity, not per individual request. A bot doing anything beyond simple reply-to-message (background polling, bulk notifications, multi-channel broadcast) needs its own request queue with backoff — otherwise a burst of activity in one part of the bot can throttle unrelated functionality elsewhere.

## Identity mapping is the quiet hard part

The Slack user ID is not the same as the identity your backend systems understand. Somewhere, a mapping has to exist between "Slack workspace + user ID" and "authenticated backend identity" — and that mapping is exactly the kind of broker problem worth designing deliberately rather than bolting on later (see the token-exchange pattern discussed in the [MCP Gateway authentication article](mcp-gateway-auth.html) — the same "broker, not vault" principle applies here).

## Mental model

> A Slackbot's hardest problems aren't the AI logic — they're the constraints of a request/response protocol that expects sub-3-second acks wrapped around an operation that takes much longer. Design for "ack fast, work async, edit in place" from day one, and most of the painful bugs (duplicate replies, broken threading, silent timeouts) never happen.
