---
layout: default
title: "RAG Retrieval Infrastructure: Chunking, MongoDB as a Vector Store, and Fixing Read Latency"
---

[← Back to index](../index.html)

# RAG Retrieval Infrastructure: Chunking, MongoDB as a Vector Store, and Fixing Read Latency

Most discussion of RAG focuses on the model side — which embedding model, which prompt, which reranker. The retrieval side is its own piece of infrastructure with its own scaling story, and it's the part that quietly determines whether "the assistant found the right passage" actually happens fast enough to matter. This covers three things from building that infrastructure on top of a single operational database used as the vector store: how chunking is actually done in practice, how a read-latency problem got diagnosed and fixed, and how index lifecycle differs for the indexes that make retrieval possible versus the ones that make everything else possible.

## Chunking is format-aware, not one-size-fits-all

The naive version of chunking is "split every document into fixed-size windows." That works poorly the moment content isn't uniform — a wide table doesn't split the same way prose does, a short self-contained record shouldn't be chunked at all, and some sources already know their own natural boundaries better than any generic splitter could guess.

**The default path handles structured text hierarchically, not by raw character count alone.** Given Markdown content, wide tables are split into sub-tables first, so a large table doesn't produce one oversized chunk. Then, if the content has headings, a header-aware splitter divides on those headings first and only falls back to a plain recursive splitter for any section that's still too big — preserving which heading a chunk fell under as metadata, so retrieval results carry their structural context along with the text. Content with no headings at all falls straight to the recursive splitter, using separators that respect code blocks and paragraph boundaries rather than cutting mid-sentence. Chunk size and overlap are both configurable, with a moderate default size and a meaningful overlap so a concept split across a boundary doesn't lose context on either side.

**Small, self-contained content skips chunking entirely.** A short record that's already a coherent unit (a small support ticket, a single Q&A pair) is stored and embedded as one chunk rather than artificially split. There's also a second, subtler lever here: the text used to *embed* something doesn't have to be identical to the text *stored and shown* to a user — trimming boilerplate (titles, repeated metadata) out of what actually goes to the embedding model measurably improves embedding quality without changing what gets displayed in a result.

**Some sources are allowed to bring their own chunk boundaries.** A generic splitter is a good default, not a mandate. A connector for chat-style content — where a single message or thread is already a complete, self-contained unit of meaning — can hand the pipeline pre-built chunks directly and skip the shared splitter altogether. The lesson generalizes: build one good default chunker, but leave an explicit escape hatch for content types where the source itself is a better judge of its own natural boundaries than a generic algorithm would be.

## One database, two workloads: MongoDB as the vector store

Rather than running a dedicated vector database alongside the primary operational store, retrieval here runs on the same MongoDB deployment that holds everything else — collections carry a vector-search index (approximate nearest neighbor, with the stored vectors quantized to shrink their footprint) side by side with a conventional full-text search index on the same content. A query combines both — dense vector similarity and lexical text matching — merged into one ranked result set, with an optional reranking pass on top for the cases where the initial ranking isn't precise enough on its own.

The appeal is real: one database to operate, one connection pool, one backup/restore story, no separate system to keep in sync with the source of truth. The cost is just as real and easy to underestimate until it actually bites: the search/ranking workload and the everyday operational read/write workload are now sharing the same underlying infrastructure, and they have genuinely different resource profiles. Ranking a query against a large vector index is CPU- and memory-hungry in a way that a typical operational read or write simply isn't. Sharing infrastructure between the two doesn't cause a problem — until retrieval volume grows enough that it does.

## Diagnosing a read-latency problem before touching anything

The way this actually got noticed wasn't a hunch — it came from watching per-namespace query performance directly: a query profiler view scatter-plots individual slow operations over time and rolls them up into a per-namespace, per-operation table (operation count, mean latency, total time spent). That view made two things visible at once: which specific collections were producing the slow reads, and that the slowness clustered — it wasn't every query getting marginally slower, it was a specific read pattern spiking hard at specific times.

That's the diagnostic step worth calling out on its own: before deciding *what* to fix, get a tool that shows you *which* queries are actually slow, on which collection, and how that changes over time — rather than reacting to a vague "the app feels slow" report and guessing.

## The fix: give search its own read capacity, not more of the shared pool

The root cause the profiler pointed at wasn't a bad query or a missing index — it was contention. Vector and text search queries were running on the same nodes that also serve the application's regular operational reads and writes, and as retrieval traffic grew, the two workloads started competing for the same CPU and memory instead of each having enough headroom of their own.

The fix was to stop treating the cluster as one undifferentiated pool of capacity: alongside the existing three-node operational replica set, two additional nodes were added specifically to serve search-index workload, separate from the base set of nodes handling ordinary reads and writes. Search and vector queries now execute against their own dedicated capacity instead of stealing cycles from the nodes an ordinary API request also depends on. The general lesson holds well beyond this one setup: once you put a search/vector workload on the same infrastructure as your operational workload, budget for the day retrieval traffic grows enough that "sharing nicely" stops being true, and know in advance that the fix is to give the search workload its own dedicated capacity, not to keep scaling the combined pool and hoping the contention resolves itself.

**Worth telling apart from this:** a search index that updates asynchronously after a write also introduces a separate, much smaller latency — typically a couple of seconds between "the write succeeded" and "a search query reflects it." That's an entirely different phenomenon (index-visibility lag on a healthy system) from the read-throughput problem above (contention under load), and conflating the two during triage sends you looking in the wrong place — one is solved by waiting or retrying once, the other by fixing capacity.

## Index hygiene: two different lifecycles for two different index types

Not every index deserves the same operational treatment, and treating them identically is itself a risk. Conventional indexes (the ones supporting ordinary lookups and filters) are cheap to build and cheap to rebuild, so it's safe to reconcile them automatically: on every service startup, compare the indexes actually defined in code against what's live on the cluster, and drop anything that's gone stale — an index nobody's code references anymore doesn't need a human to remember to clean it up.

Search and vector indexes get a deliberately more conservative lifecycle, because they're neither cheap nor safe to rebuild casually: rebuilding one takes real time, and a query arriving mid-rebuild has no business trusting a half-built index. Dropping one automatically the same way a stale conventional index gets dropped isn't a good default — that decision should be explicit (an opt-in flag, checked deliberately), and the system should actively wait for a rebuilt index to report fully ready before serving traffic against it, rather than assuming it's usable the moment it exists.

The other half of index hygiene is routine, not exceptional: when a schema change removes a field, the migration that removes the field should remove its now-unused index in the same step — a small, idempotent, dry-run-first script, not a separate cleanup task that may or may not happen later. Treating "drop the index" as part of the same change that made it unnecessary, rather than a follow-up someone might forget, is what keeps an index inventory from slowly accumulating dead weight over time.

## Closing the loop: measuring whether any of this actually helped

None of the above is worth much without a way to check whether a change (a chunking tweak, a reranking toggle, added search capacity) actually improved retrieval rather than just feeling different. The useful discipline is a small, fixed, labeled set of queries with known-relevant results, scored the same way every time: recall@k and precision@k against that labeled set, whether a model's final answer actually cites evidence that was really retrieved (not hallucinated), and — just as important as the quality metrics — p50/p95 latency for the retrieval call itself. Running that same fixed query set before and after a change (say, with reranking toggled on vs. off, or before vs. after adding dedicated search capacity) is what turns "this should be better" into a number you can actually compare.

## Mental model

> Retrieval infrastructure has the same shape as any other read-heavy system under growth: know what you're actually chunking and why the boundaries were chosen, know that a combined vector+operational database means two workloads sharing one set of resources whether you planned for that or not, diagnose slowness by looking directly at which queries are slow rather than guessing, and give a growing workload dedicated capacity before contention forces the decision on you. The index-hygiene split — reconcile freely what's cheap to rebuild, gate deliberately what isn't — is the same instinct applied to configuration instead of infrastructure.
