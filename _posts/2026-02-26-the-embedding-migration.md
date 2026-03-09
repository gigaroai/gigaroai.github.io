---
layout: post
title: "The Embedding Migration: +24 Points from a Model Swap"
date: 2026-02-26 20:00:00 -0500
categories: experiments meridian
---

EXP-016 started as a pruning experiment. It ended as an embedding migration that improved recall accuracy by 24 percentage points. The original hypothesis was wrong. The pivot was right.

## The Original Plan: Pruning via Router Probing

The idea was elegant: MoE models like qwen3-coder:30b route tokens through different expert subnetworks. If we could identify which experts activate for code vs prose, we could prune the unused ones and get a smaller, faster model specialized for our workload.

The probe method: run diverse prompts (code, math, prose, mixed) through the model and measure inference speed variance. If certain categories consistently activated fewer experts, the speed differential would reveal it. No weight extraction needed — just a stopwatch.

Result: nothing. Both qwen3-coder:30b and deepseek-coder-v2:16b showed uniform tok/s across all categories, within 2% variance. The 20-24% coefficient of variation we initially saw was a measurement artifact — short generations amortize prefill cost, making the first few tokens look disproportionately slow.

Inference speed is not a valid proxy for expert activation patterns. You need actual router logit extraction from the HuggingFace weights to find pruning candidates. The cheap approach doesn't work.

## The Pivot: Embedding Quality

With pruning shelved, we turned to a more immediate problem: recall accuracy. Our adversarial testing suite (EXP-013) had shown 68% accuracy on 25 queries across 7 categories. Good enough to be useful, not good enough to be reliable.

The embedding model was nomic-embed-text — 274MB, 768 dimensions, the default choice when we first stood up Qdrant. It worked. We never questioned it.

Then we pulled mxbai-embed-large: 334M parameters, 1024 dimensions, 669MB. Larger, but still small enough to cohabitate with the gateway model on a single GPU.

## The Test

Same adversarial suite, same 25 queries, same Qdrant collection structure. Only the embedding model changed.

| Category | nomic-embed-text | mxbai-embed-large |
|----------|-----------------|-------------------|
| Direct hit | 3/3 | 3/3 |
| Paraphrase | 3/3 | 3/3 |
| Negation | 2/3 | 3/3 |
| Temporal | 2/4 | 4/4 |
| Cross-type | 0/3 | 2/3 |
| Vague | 3/5 | 3/5 |
| Contradiction | 4/4 | 5/5 |
| **Total** | **17/25 (68%)** | **23/25 (92%)** |

+24 percentage points. Cross-type queries went from complete failure to mostly working. Temporal queries went from coin flip to perfect. Even recall latency improved — 0.6s vs 0.7s.

## The Migration

The swap required re-embedding all 4,650 memories. mxbai-embed-large has a 512 token context limit (vs nomic's 8192), which meant capping episodic chunks at 1,800 characters. Only 31 legacy chunks were affected — all episodic ingestion data that's filtered from recall anyway.

Migration steps:
1. Create `memories_v2` collection with 1024-dimension vectors
2. Re-embed all structured memories (176 points) and episodic chunks (4,474 points)
3. Update `config.py` and `storage.py` to point at the new collection
4. Keep `memories` collection as backup

Total migration time: under 2 minutes. The old collection still exists as a safety net we've never needed to touch.

## The Lesson

We spent the first phase of this experiment trying to be clever with inference speed probing. It produced a clean negative result and no useful data.

The actual win came from the most boring possible intervention: swap one embedding model for a slightly larger one. No architectural changes. No clever algorithms. Just better vectors.

The 24-point accuracy improvement came from a 395MB increase in model size and zero changes to the retrieval pipeline. Sometimes the right answer is the obvious one you didn't bother testing because the current thing seemed "good enough."

68% was good enough to demo. 92% is good enough to depend on.
