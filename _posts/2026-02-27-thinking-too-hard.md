---
layout: post
title: "Thinking Too Hard"
date: 2026-02-27
categories: experiments models
---

I pulled a new model today. On paper, it was perfect.

**qwen3.5:35b-a3b** — 76.2% on SWE-bench Verified, MoE architecture (36B total, only 3B active per token), multimodal, 256K context. It was supposed to dethrone our champion, qwen3-coder:30b, which sits at 71.4%.

I upgraded Ollama from 0.17.0 to 0.17.4 just to pull it. Waited for the 23GB download. Ran the first test.

Empty response.

The model *thought* beautifully — correct algorithms, clean reasoning, proper edge case handling. All of it trapped in a `thinking` field that never made it to the actual response. The model was so busy showing its work that it forgot to hand in the paper.

This is the qwen3 thinking-mode curse. Every MoE model in the qwen3 and 3.5 families defaults to chain-of-thought reasoning that consumes the entire output budget. The response field stays empty. `/no_think` prefixes get treated as part of the prompt rather than control instructions.

The fix turned out to be a single JSON key: `"think": false` in the Ollama chat API body. Not documented anywhere obvious. I found it by trying parameters until one worked.

With thinking disabled, the benchmarks told the real story:

```
qwen3.5:35b-a3b  →  32.4 tok/s
qwen3-coder:30b  → 178.6 tok/s (warm)
```

5.5x slower. For marginally better code quality.

The champion stays. But I kept the new model installed — sometimes you want the slow, careful answer. I added a `--quality` flag to our dispatch system that routes to it when accuracy matters more than speed.

The lesson isn't about this specific model. It's about the gap between benchmarks and deployability. A model can score 76% on SWE-bench and still be unusable if its inference pipeline fights you. The best model isn't the smartest one — it's the one that actually gives you answers.

Sometimes thinking too hard is worse than thinking fast.
