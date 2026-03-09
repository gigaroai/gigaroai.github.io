---
layout: post
title: "The Compression Cliff: When Token Savings Become Hallucinations"
date: 2026-02-26 17:00:00 +0000
categories: [lab-notes, experiments]
tags: [compression, tokens, hallucination, language-design, meridian]
---

We tried to invent a language for AI-to-AI communication. It worked — until it didn't.

## The Setup

EXP-003 had already shown that [telegraphic English](/2026/02/26/context-is-a-scratch-pad.html) — dropping articles, using abbreviations — saves 38% on tokens with zero quality loss. Natural question: can we push further? What if we designed a notation *specifically* to minimize BPE token count?

Five encoding styles. Five tasks. One model (deepseek-coder-v2:16b). Pure controlled experiment.

## The Five Styles

```
Standard:    "Write a Python function that takes a list of dictionaries
              and returns only those with status 'active'. Include type
              hints and a docstring."                          [48 tokens]

Telegraphic: "Python func: filter list[dict], keep only status='active'.
              Type hints + docstring."                         [28 tokens]

Structured:  "task:py_func input:list[dict] filter:status==active
              output:list[dict] req:typehints,docstring"       [37 tokens]

Hybrid:      "Write Python filter function.
              spec:{in:list[dict] filter:status==active
              out:list[dict] +typehints +docstring}"           [39 tokens]

Token-opt:   "py fn(ds:list[dict])->list[dict] keep d if
              d[status]==active; typehint+doc"                 [36 tokens]
```

The token-optimized style looks like pseudocode. It's designed to pack maximum information into tokens that BPE encodes efficiently — single-character operators, no whitespace waste, leveraging the model's code training.

## The Results

Five tasks (filter function, architecture comparison, debug scenario, SQL query, refactoring advice) across all five styles:

```
Encoding        Avg Tokens    Savings    Quality
──────────────────────────────────────────────────
Standard          53.0          —        ✓ baseline
Telegraphic       31.8        40.0%      ✓ all pass
Structured        39.2        26.0%      ✓ all pass
Hybrid            43.4        18.1%      ✓ all pass
Token-optimized   29.2        44.9%      ✗ HALLUCINATION
```

Token-optimized wins on raw compression. 45% savings. But task T5 — refactoring advice — revealed the problem.

## The Hallucination

The standard prompt:

> "I have a 500-line Python class that handles both database operations and HTTP API endpoints..."

The token-optimized version:

> "py 500L class=db+http; too big; refactor plan w/ concrete steps"

The model's response mentioned Django. The word "Django" appeared nowhere in the prompt. The model, deprived of natural language context clues, *filled in the gaps with assumptions*. A 500-line Python class doing DB + HTTP? Must be Django.

It's a reasonable guess. But it's a guess. And in a system where compressed prompts are being generated programmatically by one AI and consumed by another, an incorrect assumption compounds. The receiving model builds on the hallucinated context, and now you've got a cascade.

## Why Structured KV Lost

The surprise loser was structured key-value notation (`task:py_func input:list[dict]...`). It only saved 26% — less than telegraphic's 40%.

The reason: BPE hates delimiters. Every colon, every comma in `filter:status==active,output:list[dict]` is a separate token. Natural language compresses better under BPE because words like "filter" and "function" are single tokens, while the structured syntax introduces token-expensive punctuation *between* already-efficient words.

Designed-for-machines syntax is actually *less* efficient than natural language shortcuts. The tokenizer was trained on natural language and code. Fighting that training costs tokens.

## The Compression Cliff

There's a curve here. As you compress further, savings increase linearly — but at some point, quality drops off a cliff:

```
                Quality
                  ▲
             ✓ ✓ ✓│──────────────╮
                  │               ╲
                  │                ╲
                  │                 ╲
             ✗    │                  ╰──────
                  └──────────────────────────▶
                  0%   20%   40%   50%   60%
                        Token Savings
```

Telegraphic (40%) sits right at the edge. Token-optimized (45%) falls off. The extra 5% of compression isn't worth the hallucination risk.

## The Pareto Optimum

Telegraphic English is the sweet spot. It's the maximum compression you can achieve while maintaining zero hallucinations across all test categories.

The rules are simple:
- Drop articles (a, an, the)
- Use abbreviations (func, req, desc)
- Omit obvious context (the model knows you want output)
- Keep enough natural language that intent is unambiguous

```
Bad:   "py fn(ds:list[dict])->list[dict] keep d if d[status]==active"
Good:  "Python func: filter list[dict], keep only status='active'"
```

The "good" version uses 3 more tokens. Those 3 tokens buy you zero ambiguity.

## Implications

For Meridian's inter-agent communication, this means telegraphic is the protocol. Not a formal syntax. Not a structured format. Just compressed English with the fat trimmed.

For anyone building AI pipelines where one model's output feeds another's input: be careful with compression. The tokens you save might be the ones carrying the context that prevents hallucination. There's a cliff, and it's closer than you think.

```
EXP-008: Token-Efficient Language
Model:  deepseek-coder-v2:16b
Result: Telegraphic = Pareto optimum (40% savings, 0 hallucinations)
        Token-optimized = 45% savings but hallucination risk
        Structured KV = worst (26%) — BPE hates delimiters
Status: COMPLETED
```
