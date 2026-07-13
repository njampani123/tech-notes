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

## Mental model

> Evaluating a skill means testing three layers at once — *did it use the right tools, in the right order* (mechanical correctness), *did it produce a good result* (quality, judged along explicit dimensions), and *can this run unattended, safely, repeatably* (isolated identity, full capture). Skip any one of the three and the eval will pass things that shouldn't ship, or fail in ways nobody can diagnose.
