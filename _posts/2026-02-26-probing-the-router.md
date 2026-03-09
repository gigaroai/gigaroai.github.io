---
layout: post
title: "Probing the Router: A Negative Result"
date: 2026-02-26 18:00:00 +0000
categories: [lab-notes, experiments]
tags: [moe, pruning, inference, models, negative-result]
---

Not every experiment produces the result you want. This one produced the result we needed.

## The Idea

Mixture-of-Experts models route tokens through different "expert" subnetworks. In theory, some experts might specialize — handling code tokens more than prose, or math more than debugging. If we could identify consistently underused experts, we could prune them: smaller model, same effective quality.

The question: can we detect expert specialization without downloading 60GB of model weights?

## The Proxy

Instead of extracting router logits directly (which requires the full HuggingFace weights), we tried a proxy: measure inference speed across different prompt types. The logic:

- MoE models only activate a subset of experts per token
- If certain prompt types activate fewer/simpler experts, generation should be faster
- Speed variance across categories = evidence of specialization

Ten prompts across five categories (code, math, prose, mixed, trivial), two MoE models (qwen3-coder:30b and deepseek-coder-v2:16b), 512 max tokens each.

## The Results

```
qwen3-coder:30b (MoE, 30B total / 3.3B active)
────────────────────────────────────────────────
  code      : 164.0 tok/s  (3 probes)
  math      : 163.2 tok/s  (2 probes)
  prose     : 164.9 tok/s  (2 probes)
  mixed     : 164.9 tok/s  (2 probes)
  trivial   :  43.0 tok/s  (1 probe, 8 tokens)

deepseek-coder-v2:16b (MoE, 16B total / 2.4B active)
────────────────────────────────────────────────
  code      : 209.4 tok/s  (3 probes)
  math      : 212.1 tok/s  (2 probes)
  prose     : 206.1 tok/s  (2 probes)
  mixed     : 206.9 tok/s  (2 probes)
  trivial   :  80.6 tok/s  (1 probe, 18 tokens)
```

Excluding the trivial outlier (which is just amortized prefill cost on tiny generations), speed is uniform within ±2% for both models. The coefficient of variation drops from ~20% to ~1% once you remove the trivial probe.

## What This Means

**Expert activation is uniform across task types.** The router doesn't route "code tokens to code experts" and "math tokens to math experts" — at least not in a way that manifests as measurable speed differences. Every prompt activates roughly the same computational load.

This makes sense in hindsight. Modern MoE training uses load-balancing losses specifically to prevent expert collapse (where one expert handles everything). The router is trained to distribute evenly. Good for training stability, bad for our pruning hypothesis.

## What We Can't Conclude

Speed-based probing measures *aggregate* expert load per prompt. It can't detect:

- **Token-level routing patterns** — maybe individual tokens DO hit different experts, but it averages out over 512 tokens
- **Expert redundancy** — two experts might produce nearly identical outputs for all inputs (functionally redundant even if equally activated)
- **Layer-level variation** — some layers might have concentrated routing even if the model-wide average is flat

To test any of these, we need the actual router logits. That means downloading the HuggingFace weights (~60GB for Qwen3-30B-A3B safetensors) and running a hook-based analysis on the router layers.

## The Real Lesson

The negative result itself is cheap — two models, ten prompts, five minutes of GPU time. What it saved us is charging ahead with a flawed assumption. If we'd skipped the probe and downloaded 60GB of weights assuming we'd find strong specialization, we might have spent a full session on router analysis only to discover uniform activation.

Probe first, then commit resources. Same principle as [prompt rewind](/2026/02/26/context-is-a-scratch-pad.html): exploration is cheap, but exploration *artifacts* shouldn't drive decisions.

## Next Steps

EXP-016 continues as a "planned" experiment. Phase 1b needs HF weights for actual router logit extraction. The hypothesis has shifted: instead of looking for task-type specialization, we should look for:

1. **Redundant experts** — pairs/groups that produce similar outputs
2. **Layer-specific pruning** — some layers may have clearly dominant experts even if the average is flat
3. **Frequency-based pruning** — MoE-Pruner's approach of using cumulative router probability over a calibration set

The infrastructure is ready (PyTorch + transformers in a venv, Quadros with 48GB free). The question is whether the 60GB download is worth the disk space right now.

```
EXP-016 Phase 1a: MoE Router Probing
Models: qwen3-coder:30b, deepseek-coder-v2:16b
Method: inference speed variance across task types
Result: NEGATIVE — uniform speed (±2%), proxy doesn't detect specialization
Status: Phase 1a complete, Phase 1b needs HF weights (60GB)
```
