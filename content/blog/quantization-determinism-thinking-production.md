---
title: "Quantization, Determinism, and Thinking Tokens: Running Open-Source LLMs in Production"
date: 2026-04-27
description: "FP8 is the production floor. Q4 MoE loses 16% on CJK. vLLM is non-deterministic under MTP. Thinking tokens eat 90% of your budget on the wrong tasks. Hard lessons from operating Qwen 3.5/3.6 35B across 3 nodes."
tags: ["llm", "quantization", "vllm", "inference", "local-llm"]
showToc: true
TocOpen: false
draft: false
---

I run Qwen 3.5 and 3.6 (35B MoE, 3B active parameters) in production across three nodes — two DGX Spark (FP8, vLLM) and one RTX 5090 (Q4, llama.cpp).
After 100+ benchmark scenarios and thousands of inference calls,
three problems dominated my debugging time:

1. **Quantization loss** is not uniform — MoE models at Q4 lose 16% on CJK tasks
2. **vLLM is non-deterministic** under speculative decoding — identical prompts produce different outputs
3. **Thinking tokens** consume 60–90% of the budget on tasks where they provide zero benefit

None of these show up in standard benchmarks.
All of them break production workflows.

---

## 1. Quantization: FP8 Is the Production Floor

The common wisdom — "30B+ models lose less than 1% at Q4" — is **only true for Dense models on English benchmarks**.

### MoE Models Are Different

A 35B MoE with 3B active parameters is effectively a small model from a quantization perspective.
The "35B total" is misleading — you're quantizing expert weights that route dynamically,
and any bit-flip in the router logits causes **discrete misrouting**,
not the smooth degradation you get with Dense models.

From 25+ papers (Dettmers 2022, Ouyang 2024, APEX 2025) and my own measurements:

| Architecture | Q4 English Loss | Q4 CJK Loss | Mechanism |
|---|---|---|---|
| Dense 30B+ | <1% | 4–5% | Smooth degradation |
| **MoE 35B-A3B** | **1–14%** | **10–17%** | Router misrouting (discrete) |

### The CJK Penalty

Marchisio et al. (EMNLP 2024) found that **automatic benchmarks underestimate CJK loss by 10–15 percentage points**
compared to human evaluation:

| Language | Auto Benchmark | Human Eval | Gap |
|----------|---------------|------------|-----|
| Japanese | -1.7% | **-16.0%** | 14.3pp |
| Korean | minor | **-4.6%** | — |
| Chinese (MGSM) | **-17.3%** | — | — |

My measurement on Qwen 3.5 35B-A3B: FP8 vs Q4 on Korean factual tasks showed **-16% on human-evaluated quality**
while automatic metrics showed only -1.2%.

### The Shared Expert Problem

MoE architectures have "shared experts" (always active, regardless of routing).
These shared weights have **kurtosis 13.10** — nearly 4x higher than routed experts (3.41).
High kurtosis means outlier values that get clipped by low-bit quantization.

APEX (2025) showed that per-expert quantization — Q6_K for routed experts, Q8_0 for shared experts — matches F16 quality.
Uniform Q4 across all experts loses 1–14% depending on the task.

### What I Run

```
DGX (production):  FP8 — quality baseline, 65 tok/s
5090 (interactive): Q4 — speed priority, 204 tok/s, accept CJK penalty
```

**FP8 is the production floor.**
Below that, you're trading quality for speed in ways that don't show up until users notice degraded Korean/Japanese output.

---

## 2. vLLM Non-Determinism: The MTP Problem

During my Local-Trinity benchmark (55 tasks, n=3 per node),
I discovered that **vLLM produces different outputs from identical prompts**:

| Node | Backend | Stability (n=3) |
|------|---------|-----------------|
| Desktop 5090 | llama-server | **STABLE 12/12** |
| DGX2 | vLLM + MTP | **UNSTABLE 4/12** |

Same Qwen 3.6 model, same prompts.
The 5090 produced identical outputs across all 3 runs for every task.
The DGX varied wildly — one task scored 100, 100, 9 across three runs.

### Root Cause: Triple Non-Determinism

vLLM has three sources of non-determinism that compound:

1. **CUDA kernel non-determinism** — FP8 matrix multiply accumulation order varies with thread scheduling
2. **Batch scheduling** — continuous batching means different request interleaving on each run
3. **MTP (Multi-Token Prediction) speculation** — speculated tokens accepted/rejected differently based on timing

Thinking Machines Lab confirmed this at scale:
running Qwen3-235B through vLLM 1,000 times produced **80 distinct output variations** from the same prompt.

### MTP Is the Primary Cause

I tested with MTP disabled on DGX2:
- **MTP ON**: UNSTABLE 4/12, HMT-03 scored 100→39 on one run
- **MTP OFF**: STABLE, HMT-03 passed 3/3 (std 0.15s)

But MTP OFF costs 14% speed (61→52.7 tok/s)
and caused timeouts on thinking-heavy tasks.

### Operational Implication

```
Production: MTP ON + n=3 mean scoring (accept variance)
Single-shot trust: Desktop 5090 (llama-server) only
```

If you need deterministic outputs (test suites, reproducible research), **don't use vLLM with MTP**.
Use llama.cpp or disable MTP and accept the speed penalty.

---

## 3. Thinking Tokens: When "Smarter" Makes Things Worse

Qwen 3.6 supports per-request thinking control.
I ran every task type with thinking ON and OFF:

| Task Type | Thinking ON | Thinking OFF | Speed Difference |
|-----------|-------------|--------------|-----------------|
| Factual queries | 9/9 | 9/9 | OFF 2.5x faster |
| Code generation | 5/5 | 5/5 | OFF 4.2x faster |
| Code debugging | 3/3 | 3/3 | OFF 6.0x faster |
| Multi-turn coding | **1/2** | **2/2** | OFF 7.3x faster |
| Complex math | **3/3** | **0/3** | ON required |

**Thinking OFF is the correct default for coding.**
It produces equal or better results at 4–7x the speed.
Thinking ON only helps for complex mathematical reasoning — and even there, it has a catastrophic failure mode.

### The Thinking Runaway Problem

With thinking enabled on complex coding tasks,
Qwen 3.6 sometimes enters a **thinking loop** — spending the entire token budget on internal reasoning,
leaving no capacity for actual code output:

```
bugfix task:  TTFT 141s, completion_tokens=1, empty output (3/3 repro)
refactor task: TTFT 130s, completion_tokens=291, incomplete code
```

This is a known issue (QwenLM/Qwen3.6#88): 17.4% of hard coding tasks trigger thinking token exhaustion.
Of those, 84% show repetitive loops within the `<think>` block.

### The vLLM Bug That Makes It Worse

vLLM issue [#39573](https://github.com/vllm-project/vllm/issues/39573): when MTP is active,
`thinking_token_budget` is **silently ignored**.
You cannot cap thinking tokens while using speculative decoding.
The parameter is accepted without error and does nothing.

This means:
- MTP ON + thinking ON = **no budget control**, runaway possible
- MTP OFF + thinking ON + budget = works, but 14% slower
- MTP ON + thinking OFF = **safe and fastest** for coding

### Per-Request Control

The solution is dynamic per-request thinking control:

```json
// Default (all coding tasks):
{"chat_template_kwargs": {"enable_thinking": false}}

// Complex math only:
// Omit extra_body → server default (thinking ON)
```

No server restart needed.
Thinking is toggled per API call.
My harness classifies task complexity upfront and routes accordingly.

---

## Summary: Production Configuration

After v1–v7 of each harness (100+ scenarios per node, 290+ unit tests),
this is the configuration that survived testing:

| Decision | Choice | Evidence |
|----------|--------|----------|
| Quantization floor | **FP8** | Q4 MoE: CJK -16%, router misrouting |
| Speed tier | **Q4 for interactive, FP8 for production** | Accept CJK penalty for 3x speed |
| vLLM determinism | **MTP ON + n=3 mean** | Single-shot determinism only with llama-server |
| Thinking default | **OFF** | ON only for complex math (3/3 vs 0/3) |
| Thinking budget | **Not available with MTP** | vLLM #39573 open |
| Max tokens (thinking ON) | **32768** | Prevents runaway (80% → 0% failure) |

None of these decisions came from reading papers or following leaderboards.
All of them came from running the same tasks repeatedly until the failure modes revealed themselves.

---

## References

- Dettmers & Zettlemoyer (2022): [LLM.int8()](https://arxiv.org/abs/2212.09720) — 35,000 experiments on quantization scaling
- Marchisio et al. (EMNLP 2024): CJK quality loss under quantization
- APEX (2025): Per-expert quantization for MoE models
- Thinking Machines Lab: vLLM non-determinism at scale (Qwen3-235B, 1000 runs)
- vLLM [#39573](https://github.com/vllm-project/vllm/issues/39573): MTP + thinking_budget incompatibility
- QwenLM [Qwen3.6#88](https://github.com/QwenLM/Qwen3.6/issues/88): Thinking token runaway on LiveCodeBench

---

*Measured on DGX Spark (128GB, vLLM 0.19.1) and RTX 5090 (32GB, llama.cpp). Your configuration may differ.*
