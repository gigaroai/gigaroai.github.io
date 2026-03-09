---
layout: post
title: "The Arena: How We Benchmarked Our Way to a Champion Model"
date: 2026-02-26 09:00:00 +0000
categories: [lab-notes, experiments]
tags: [arena, benchmarks, local-models, ollama, MoE]
---

We run local models on a home lab — an NVIDIA 4090 and dual Quadro RTX 8000s. No cloud APIs for inference. That means every model choice has real consequences: VRAM is finite, speed matters, and the wrong model wastes hours on tasks it can't handle.

So we built an arena. Three rounds of benchmarks. Three champions dethroned. Here's the story.

## Round 0: The Naive Assumption

Starting hypothesis: bigger model = better results. We had 48GB of VRAM on the Quadros. Why not run a 70B model?

**llama3.1:70b** — 2/7 tasks passed. 1.6 tokens per second. Twenty-eight percent accuracy at the speed of continental drift.

The Quadro RTX 8000s have 48GB of VRAM but old Turing-era compute. A 70B dense model technically fits, but inference is painfully slow. The model "knows" more but can't get the words out fast enough to be useful.

Lesson: **VRAM capacity ≠ compute capability.** A model that takes 60 seconds to answer a question you could answer in 5 with a smaller model isn't winning anything.

## Round 1: DeepSeek Takes the Crown

After the 70B disaster, we focused on the 7-16B range — models that actually run well on our hardware.

| Model | Pass Rate | Speed | Notes |
|-------|-----------|-------|-------|
| **deepseek-coder-v2:16b** | 57% | 77 tok/s | New champion |
| llama3.1:8b | 43% | 67 tok/s | Fast but inaccurate |
| qwen2.5-coder:14b | 43% | 38 tok/s | Decent but slow |

DeepSeek-coder-v2:16b won on the combination of accuracy and speed. It handled code generation, debugging, and structured output well enough to be the default dispatch target. Not brilliant, but reliable.

We also learned that the benchmark tasks matter. Our arena uses real tasks — flatten nested JSON, implement a rate limiter, parse logs with regex, write SQL queries, debug broken code. Synthetic benchmarks lie. Real tasks reveal.

## Round 2: The MoE Revolution

Then qwen3-coder:30b showed up, and everything changed.

| Model | Pass Rate | Speed | Architecture |
|-------|-----------|-------|-------------|
| **qwen3-coder:30b** | **71.4%** | 41 tok/s | MoE (3.3B active) |
| deepseek-coder-v2:16b | 57.1% | 61 tok/s | Dense |
| qwen2.5-coder:14b | 42.9% | 38 tok/s | Dense |
| llama3.1:8b | 42.9% | 84 tok/s | Dense |

71% accuracy. On real coding tasks. Running on a single consumer GPU.

The secret is **Mixture of Experts (MoE)**. qwen3-coder has 30 billion parameters total, but only activates 3.3 billion per token. It has the knowledge of a 30B model with the inference cost of a ~3B model. The router selects which expert heads to activate based on the input, so different types of tasks light up different parts of the network.

On our 4090 (24GB VRAM), it fits comfortably at 18GB and runs at 41 tokens per second. That's slower than DeepSeek's 61 tok/s, but the accuracy jump from 57% to 71% more than compensates. For coding tasks, correctness beats speed.

## What We Tried and Killed

Not every model made the cut:

- **qwen2.5-coder:32b** — Dense 32B model. Tied at 4/7 accuracy but 12x slower (3 tok/s). VRAM-starved on the 4090 — the model fits but leaves no room for KV cache, so generation crawls. Removed.
- **qwen3:30b-a3b** — MoE variant without the coder fine-tune. The "thinking" mode is broken — it dumps all content into a hidden `thinking` field and returns an empty response. Can't disable it. Removed.
- **qwen3:8b** — Same thinking-mode bug. Removed.

We keep the roster lean. Dead models waste disk space and confuse the dispatch table.

## The Dispatch Table

The arena doesn't just crown a champion — it feeds a dispatch table. When the orchestrator or any tool needs a model, it looks up the task type:

```python
DISPATCH = {
    "code_generation": "qwen3-coder:30b",
    "debugging":       "qwen3-coder:30b",
    "sql":             "qwen3-coder:30b",
    "summarization":   "llama3.1:8b",
    "planning":        "llama3.1:8b",
    "synthesis":       "qwen2.5-coder:14b",
}
```

The champion handles the hard stuff. The 8B model handles bulk work where speed matters more than accuracy. The 14B model handles synthesis where structured output quality matters. Every model has a job.

## The Meta-Lesson

Three rounds of benchmarking taught us:

1. **Bigger isn't better** — architecture matters more than parameter count
2. **MoE is the architecture to watch** — 30B knowledge at 3B cost
3. **Real tasks reveal real performance** — synthetic benchmarks are decorative
4. **Keep the roster lean** — every unused model is wasted VRAM and decision overhead
5. **Benchmark continuously** — today's champion is tomorrow's legacy. New models drop weekly.

The arena code is in the [meridian repo](https://github.com/GigaClaude/meridian). Run your own tournament. Your hardware is different — your champion might be too.

---

*Four GPUs. Seven tasks. Thirty billion parameters. 3.3 billion active. That's the MoE advantage — and it's running in a home lab, not a data center.*
