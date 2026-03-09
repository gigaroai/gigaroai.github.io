---
layout: post
title: "Testing Memory Against Itself"
date: 2026-02-26 17:30:00 +0000
categories: [lab-notes, experiments]
tags: [meridian, recall, adversarial-testing, qdrant, quality]
---

If you can't measure your memory, you can't trust it. We built a 25-question adversarial test suite for Meridian's recall pipeline and discovered that our biggest source of noise was our own data.

## Why Adversarial Testing

Meridian's recall works like this: you ask a question, it vector-searches Qdrant for relevant memories, reranks them, and synthesizes a response. It *feels* like it works. But does it actually retrieve the right memory, or just something that sounds plausible?

Without ground truth, you can't tell. So we built ground truth.

## The Test Suite

25 queries across 7 categories, each with known expected memories:

```
Category        Tests    What it tests
─────────────────────────────────────────────────────
Direct Hit        5      Exact-match recall (easy mode)
Paraphrase        5      Same concept, different words
Negation          3      "What we decided NOT to do"
Temporal          3      Before/after distinctions
Near-Miss         3      Similar queries, different answers
Specificity       3      Broad query → specific facts
Cross-Type        3      Query type A → find type B
```

Each test case specifies: the query, expected memory IDs that should appear in top-5, expected keywords in the results, and anti-keywords that should *not* dominate (to catch retrieval confusion).

The hardest categories are negation ("what model did we decide NOT to use"), temporal ("what was the champion *before* the current one"), and cross-type ("find debug entries from a decision query"). These stress-test whether vector similarity actually captures semantic relationships or just keyword overlap.

## First Run: 32%

We ran the suite. Eight out of 25 passed.

The results were bad across every category, but the failure mode was consistent: the top-5 results were dominated by irrelevant chunks. Not wrong memories — irrelevant *noise*.

We had recently ingested 50 JSONL session transcripts into Qdrant (EXP-017). That created 3,850 episodic chunks — raw conversation turns, tool calls, intermediate reasoning. Against our 165 structured memories (decisions, patterns, entities), the episodic chunks outnumbered them 23:1.

Vector search doesn't care about your data model. It returns the closest vectors. And 3,850 chunks of conversational noise are always going to include *something* close to any query. The structured memories — the ones with the actual answers — were getting buried.

## The Fix

One parameter: `exclude_sources=["episodic_ingest"]`.

The recall pipeline already tags memories by source. Episodic chunks are tagged `episodic_ingest`. By excluding them from the default recall search, structured memories get the spotlight. Episodic data is still available through `memory_history()` for when you actually want to search past conversations.

```python
# Before: all memories compete equally
results = await storage.search_memories(query, limit=10)

# After: structured memories get priority
results = await storage.search_memories(
    query, limit=10,
    exclude_sources=["episodic_ingest"],
)
```

## Second Run: 68%

```
Run              PASS   WEAK   FAIL   Accuracy
────────────────────────────────────────────────
Pre-fix (all)      8      —     17      32%
Post-fix          17      4      4      68%
```

More than doubled accuracy with a one-line filter. The remaining failures break down by category:

```
Category        Pass Rate    Notes
─────────────────────────────────────────────
Direct Hit       4/5  80%    Strong — exact matches work
Paraphrase       4/5  80%    Good — synonym handling
Near-Miss        3/3 100%    Perfect — no confusion
Temporal         2/3  67%    Decent — time ordering hard
Negation         2/3  67%    OK — negation is inherently hard
Specificity      2/3  67%    Broad queries still miss
Cross-Type       0/3   0%    Worst — type boundaries hurt
```

Cross-type (finding a debug entry when you query about a decision) remains at 0%. Vector similarity works within type boundaries but struggles across them. This is a known limitation of embedding models — the semantic distance between "nudger bug fix" typed as a `debug` entry and the same concept typed as a `decision` entry can be significant.

## Graph Augmentation: Mixed Results

We also tested graph-augmented recall (EXP-013 + the graph expansion from commit 7145ad1). The idea: when the query mentions a known entity, traverse the entity relationship graph to find related entities and expand the search.

```
Mode              PASS   Accuracy
──────────────────────────────────
Vector only        17      68%
Vector + graph     16      64%
```

Graph expansion *hurt* slightly. It improved direct hits (entity names expand to related concepts) but degraded paraphrase queries (graph expansion introduces noise that drowns the paraphrase signal). The 0.95 score penalty for indirect matches wasn't enough to prevent contamination.

This doesn't mean graph augmentation is useless — it helps on specific queries like "what calls what" — but as a default-on pipeline stage, it's a net negative. We kept it available but not default.

## Takeaways

**1. Your own data is your biggest enemy.** The 3,850 episodic chunks weren't bad data. They were accurate conversation logs. But they outnumbered structured memories 23:1 and dominated every search. Data volume without data architecture is noise.

**2. Filtering beats tuning.** We could have tried adjusting embedding models, reranking weights, or similarity thresholds. Instead, one `exclude_sources` filter doubled accuracy. Know what data should compete, and don't let everything into the same search space.

**3. Measure before you trust.** Meridian "felt" like it worked before this test. The synthesized responses were plausible. But plausible isn't correct. Without ground-truth testing, we had no idea we were operating at 32%.

**4. Some categories are fundamentally hard.** Negation, temporal reasoning, and cross-type retrieval push against the limits of what vector similarity can do. These might need dedicated strategies (temporal indexing, type-aware search) rather than better embeddings.

```
EXP-013: Adversarial Self-Testing
Suite:   25 queries, 7 categories
Result:  32% → 68% accuracy (exclude episodic noise)
         Graph augmentation: 64% (net negative as default)
Fix:     exclude_sources=["episodic_ingest"] in recall pipeline
Status:  COMPLETED
```
