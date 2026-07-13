---
layout: default
title: "Evaluating Skills: A Methodology for Testing AI-Assistant Skills End-to-End"
---

[← Back to index](../index.html)

# Evaluating Skills: A Methodology for Testing AI-Assistant Skills End-to-End

Testing a [skill](understanding-skills.html) isn't like testing a normal function — a skill mixes natural-language instructions, tool calls, and often generated content, and "correctness" is frequently a quality judgment rather than an exact match. That calls for an evaluation methodology, not just unit tests.

## Step 1: Load and wire the skill

Before running any prompts, resolve the skill's definition and connect whatever tool servers it declares as dependencies. Then explicitly verify that the tools the skill calls out are actually **present and loaded into the assistant's context** — not just that the connection succeeded. A misconfigured or unreachable tool server should fail loudly right here, at setup time, rather than surfacing as a confusing mid-conversation failure later in the eval run.

## Step 2: Use an isolated test identity

Run evals under a **dedicated, sandboxed test account**, not a real user's credentials. Interactive/manual auth steps (device approval, two-factor prompts) should be disabled for this account specifically, so headless, automated runs can complete without a human in the loop — while keeping the account scoped so a failure or bug during the eval can't touch real user data or production systems.

## Step 3: Run a prompt suite and capture everything

Build a set of representative prompts covering the skill's intended use cases and known edge cases. For every run, capture three things together as one record:

- the **input** (prompt, and any supplied context/files),
- the full **tool-call sequence** the assistant actually executed,
- the final **output**.

Capturing all three — not just the final output — is what makes failures debuggable later; an output-only log can't tell you *why* something went wrong.

## Step 4: Verify tool sequencing, not just the final output

A good-looking final output produced through the wrong tool path is still a bug. Check that the sequence of tool calls matches the expected plan for that prompt — right tools, right order. Common failure patterns this catches: skipping a validation step that should have run first, calling a destructive or write-capable tool when a read-only one would have sufficed, or reaching the correct answer by accident via an unintended path that won't generalize.

## Step 5: Score with an LLM judge, across explicit dimensions

Because much of a skill's output is generated content, a second model acts as judge — but scored along **separate, explicit dimensions** rather than one blended score, since a single number hides *which* aspect actually failed. Dimensions should fit what the skill actually does; common ones include:

- **Instruction adherence** — did the output actually do what was asked?
- **Integrity/faithfulness** — did it stay true to the source material or subject, without distorting or hallucinating details?
- **Quality** — is the output actually fit for its intended purpose?
- **Safety** — is the output policy-compliant, free of harmful or disallowed content?

Scoring each dimension independently means a regression in one area (say, safety) doesn't get averaged away by strong scores elsewhere.

## Step 6: Aggregate and gate

Roll the per-run scores up per skill (and per skill version), and use the aggregate as a **release gate** — a skill update that regresses on any dimension should be caught before shipping, not after. When a run fails, the captured input/tool-sequence/output triple from Step 3 is what makes the failure actually debuggable rather than just a red X in a dashboard.

## Step 7: Trace every run for observability

Aggregate pass/fail scores tell you *that* something regressed; they don't tell you *why*, and they're not shareable evidence. Sending every prompt, input, tool-call sequence, and output to a **tracing/observability platform** turns each eval run into an inspectable record rather than a single number. This matters most when a failure isn't the skill's fault at all — if a specific tool is returning malformed or empty results, a trace is exactly what you hand to the team that owns that tool, instead of describing the failure secondhand.

## Auto-classifying failures at scale

Once evals run continuously (on every change, or on a schedule), failures accumulate faster than any human can triage one by one. The first thing worth automating isn't root-causing — it's **sorting failures into buckets** before a human ever looks at them:

- **Authentication error** — the call never reached real logic; the test identity's credentials failed or expired.
- **API/infra error** — a rate limit, timeout, or 5xx from a backend the tool depends on; often transient, not a real regression.
- **Tool error** — the tool ran and returned a result, but the result itself is wrong, empty, or malformed.
- **Quality/judge failure** — everything executed correctly, but the LLM judge scored the output poorly on one of the eval dimensions.

These categories are usually distinguishable from signals already available at the point of failure (status codes, exception types, whether the judge ran at all) — so this classification can be automatic, not manual. Automating it is what makes continuous evaluation sustainable, since it turns "figure out what kind of failure this is" from a per-incident human task into a one-time rule.

## Ownership via a tool registry

When tools are built and owned by different teams, "whose bug is this" becomes its own bottleneck — especially once failures are already sorted into a "tool error" bucket but nobody agrees who should pick it up. A **tool registry** that maps every tool to its owning team turns this from a discussion into a lookup: once a failure is classified as a tool error for a specific tool, the registry says exactly where it routes, and the traced record (Step above) is what gets attached when it's handed off.

## Challenges and how to address them

Running evals continuously, at scale, surfaces problems that don't show up in a one-off test pass:

- **Failure volume outpaces manual review.** Continuous runs across many prompts and skill versions generate more failures than a team can individually inspect. *Address it by deduplicating first* — group failures by a root-cause signature (same tool, same error category, similar context) before a human sees them, so a hundred raw failures might collapse into a handful of distinct underlying issues to actually review.
- **Not every failure is a real regression.** Transient rate limits, flaky infrastructure, or judge-model scoring noise all produce failures that look like regressions but aren't. *Address it by tracking flakiness per test case over time* — a prompt that fails intermittently across many runs without a code change is a candidate for quarantine/flagging, not repeated manual re-litigation.
- **Auto-triage rules go stale.** As tools and failure modes change, a fixed classifier will eventually misfile new failure types into the wrong bucket. *Address it by feeding manual review findings back into the classifier* — every failure a human has to manually categorize is a signal that the auto-triage rules are missing a case, and should be closing that gap over time rather than accumulating the same manual work indefinitely.
- **Ownership boundaries blur when tools interact.** A failure might trace back to how two teams' tools were composed together, not either tool in isolation — the registry can name an owner for a single tool, but not always for an interaction between tools. This case generally still needs a human judgment call, and is worth explicitly tracking as its own (smaller) category rather than forcing it into either team's bucket.

## Mental model

> Evaluating a skill means testing three layers at once — *did it use the right tools, in the right order* (mechanical correctness), *did it produce a good result* (quality, judged along explicit dimensions), and *can this run unattended, safely, repeatably* (isolated identity, full capture). Running that evaluation *continuously* adds a fourth layer — *can failures be triaged and routed faster than they accumulate* — and that layer is solved with the same instinct as the others: capture everything, classify automatically wherever the signal allows it, and reserve human judgment for the cases that genuinely need it.
