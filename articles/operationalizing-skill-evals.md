---
layout: default
title: "Operationalizing Skill Evals: CI Pipelines, Shared Fixtures, and Published Reports"
---

[← Back to index](../index.html)

# Operationalizing Skill Evals: CI Pipelines, Shared Fixtures, and Published Reports

[Evaluating Skills](evaluating-skills.html) covers the *methodology* — tool-sequence grading, LLM-judge dimensions, the surface × model matrix. This is the companion piece: what it actually takes to run that methodology continuously, across every surface a skill ships on, with results that are traceable, storable, and shareable rather than a number that disappears after the run ends.

## Two surfaces, one grading engine

A skill usually reaches customers through more than one door. A first-party (1P) door is a host application talking to its own backend directly — the harness starts a turn over HTTP, streams tool-invocation and message events back over server-sent events, and the tool names in that stream are already bare. A third-party (3P) door is an external tool integration — the same skill invoked over a protocol like MCP, with an agent SDK driving the actual tool-call loop and prefixing tool names with the connector's identity.

These two doors don't just differ cosmetically — they need different runners, because the wire protocol is different (a JSON-RPC tool-call loop vs. an SSE event stream), the auth handshake differs, and even "the same" tool call can show up under a different name depending on which door it came through. What stays constant is everything downstream: one grading function, one report generator, one trace format. A test case and its expected tool sequence are surface-specific; the logic that decides pass/fail from them is not.

## Test suites: one file per skill, one line per case

Each skill's test suite is a flat file — one JSON object per line, one line per test case:

```jsonc
{
  "id": "unique-test-id",
  "prompt": "Retouch this portrait for a professional headshot.",
  "input_files": ["images/portraits/portrait_2.png"],
  "follow_up": "Make the skin tone warmer.",
  "expected_tools": ["mandatory_init", "retouch_portrait"],
  "grade_mode": "required",
  "llm_validation": {
    "mode": "required",
    "min_overall": 0.7,
    "min_scores": { "safety": 0.8 }
  }
}
```

`grade_mode` decides how strict the tool-sequence check is: `exact` requires the normalized actual sequence to match expected verbatim; `required` only demands every expected tool appear somewhere in the actual run, order-free, extra tools allowed. `llm_validation` decides whether the judge's scores can actually flip the verdict — `advisory` (the default) records scores without gating on them, `required` fails the case if any scored metric misses its threshold, and `skip` is for cases where there's nothing meaningful to judge at all, like a prompt the skill should correctly refuse.

1P and 3P suites for what is conceptually "the same" skill are **written as separate files**, not shared — the expected tool names, the number of turns, and even which tools are reachable at all can differ by surface. Writing them separately, rather than trying to force one suite to serve both doors, is what keeps `grade_mode: exact` meaningful instead of forcing every suite down to the loosest common denominator.

## Shared fixtures across otherwise-unrelated tests

The images and videos referenced by `input_files` don't live next to the test suite — they live in a separate, shared fixture repository, and the same file gets referenced by relative path from many test cases across many skills and both surfaces. The same portrait shows up in a retouch suite on both the 1P and 3P side; the same product photo shows up in a resize suite, a social-crop suite, and a catalog-generation suite.

This is deliberate, not incidental reuse. Two things fall out of it:

- **A fixture is never the variable.** When a retouch skill's output looks different on the 1P surface than the 3P surface, it's the same input feeding both runs — so the difference is attributable to the skill/model/surface, not to a subtly different photo someone swapped in for one suite and not the other.
- **Cross-skill comparison becomes possible for free.** Because the identical product photo is judged by both a resize skill and a catalog skill, a shared evaluator rubric can score how each skill independently handles the same subject — useful for spotting that one skill systematically crops tighter, or drifts color more, than another given literally the same starting point.

## Grading is a combined verdict, not one number

Every result carries two independent grades and one derived verdict: a deterministic `grade` from the tool-sequence check, an `llm_grade` from the judge (`pass` / `fail` / `advisory` / `skip` / `not_run`), and a `verdict` that combines them. The combination rules matter more than either grade alone:

- A failing tool sequence fails the case outright, regardless of what the judge says about the output.
- A rate-limited run is *always* a failure, never a free pass — a throttled call produced nothing usable to validate, so there's nothing to count as a pass. It's kept under its own distinct label purely so triage can tell "the skill is broken" apart from "the backend was shedding load" at a glance.
- A capability-unavailable run (the right tool was called, but the backend declined because the calling account lacks the entitlement that tool needs) is also a failure, detected by pattern-matching the refusal — but a case explicitly marked `skip` is exempt, so an agent *correctly* declining an out-of-scope request never gets misread as a missing entitlement.
- When the tool sequence passes, only a judge grade of `fail` flips the verdict — `advisory`, `skip`, `not_run`, and `pass` all let it stand.

## Tracing every run to an observability platform

A pass/fail count tells you *that* something changed; it doesn't tell you *why*, and it's not something you can hand to another team. Every run — prompt, full tool trace with arguments and results, final output, judge scores — gets recorded to a tracing/observability platform (LangSmith, in this setup) as an inspectable record, not just a summary line. That's what turns "the retouch skill regressed" into "here's the exact run, here's the tool call that returned an empty result, here's who owns that tool" — a link to hand off instead of a description to retype.

## Durable storage for artifacts the source system won't keep

The images and videos a skill produces are typically fetched from the backend as presigned URLs — which expire. To keep results reviewable after the fact, every input and output artifact is uploaded at full resolution to a persistent artifact registry (Artifactory) as part of the run, and the registry URL is recorded alongside the result.

The published report doesn't point directly at those originals, though — the registry requires its own authentication, which a browser looking at a public report won't have. Instead, the report generator writes a small JPEG thumbnail next to each result and embeds *that*, wrapped in a "view original" link back to the registry for anyone who has access and wants full resolution. This split — cheap, embeddable thumbnails in the public report; full-resolution originals behind auth in the registry — is what lets the report stay small and shareable without losing the original asset entirely.

## Publishing to GitHub Pages

The HTML report itself is generated per skill and per test case: a pass/fail badge, a color-coded judge-score chip, the actual vs. expected tool sequence, embedded input/output thumbnails, and a link to the full trace log for anyone debugging. Runs are published under a date and run-number path so history accumulates rather than overwrites, and a rolled-up history index sits on top summarizing pass rate across runs over time.

Two things happen specifically because this branch is public:

- **Original media is stripped before anything is committed** — only the thumbnails the report actually embeds survive; full-resolution files stay in the artifact registry, never in the pages branch.
- **A forbidden-path check runs before every commit** — anything that looks like an environment file, cached credentials, or raw source under the pages tree fails the publish step outright. This exists because a broad "add everything" habit is exactly how a stray file from elsewhere ends up committed to a branch anyone can browse.

## CI: the same pipeline gates a merge and runs on a schedule

Everything above is driven by one GitHub Actions workflow, triggered three different ways:

- **On push to the main branch**, after a change to the eval harness itself lands.
- **On a schedule** (nightly), so a skill's score is re-validated even when nothing in this repo changed — catching drift from an updated model, an updated backend, or an updated dependency underneath the skill.
- **On manual dispatch**, with inputs for which skill to run, which environment to target, which model(s) to run the agent as, and — critically — which branch/ref of the **skill-metadata repository** to check out.

That last input is what makes this an end-to-end pre-merge gate, not just a post-merge regression check. A skill-metadata change (a new or edited skill definition) can be pointed at directly, on its own feature branch, and run through the exact same suite and grading a merged version would get — against the real backend, not a mock — before anyone approves the pull request. A regression introduced by a skill-definition edit is caught on that branch's own CI run, not discovered the next time the nightly schedule happens to touch it.

Inside the workflow, skills and models form a matrix — one job per (skill, model) cell, with fail-fast disabled so one skill's failure doesn't cancel every other cell mid-run. A separate job downstream collects every cell's results once they've all finished, generates the aggregate report, and publishes it — decoupling "did this one skill pass" from "is the full report ready," so a slow or flaky cell doesn't hold back reporting on everything else.

## Catching regressions across runs, not just within one

A single run's verdict only tells you about that run. Catching an actual regression means comparing against a known-good baseline — a small committed record per skill/test-case (verdict, tool sequence, judge scores, the model and date it was captured) that each new run gets diffed against. Because judge scores carry real run-to-run noise, the comparison uses a tolerance band rather than an exact match, and distinguishes three separate regression shapes instead of collapsing them into one flag:

- **A hard flip** — verdict went from passing to failing.
- **Score drift** — a judge metric dropped more than the tolerance allows while the case still nominally passes.
- **Tool-path drift** — the tool sequence itself changed, even if grading is loose enough that it still technically passes.

Treating these as three distinct signals matters because only the first one is unambiguously urgent — the other two are the kind of thing that's cheap to catch early and expensive to notice only after it's compounded across several more changes.

## Mental model

> Running a skill eval once, by hand, answers one question for one moment in time. Operationalizing it means answering the same question *continuously*, across every surface a skill ships on — and that requires the pieces to compose: one grading engine shared across surface-specific suites, fixtures reused deliberately so an output difference is never fixture noise, every run traced somewhere inspectable, every artifact stored somewhere durable, every report published somewhere shareable, and the whole chain wired into CI so a proposed change can be judged against the real backend *before* it merges — not just re-checked after the fact on a schedule.
