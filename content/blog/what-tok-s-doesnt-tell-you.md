---
title: "What tok/s Doesn't Tell You: Measuring LLM Speed That Matters"
date: 2026-04-24
description: "My 204 tok/s GPU feels slower than a 65 tok/s one for some tasks. tok/s alone is a misleading metric — here's a framework (TTR, Effective tok/s, TCT) that measures what users actually experience."
tags: ["llm", "benchmarks", "inference", "local-llm"]
showToc: true
TocOpen: false
draft: false
---

I run Qwen 3.6 35B on three machines.
The RTX 5090 generates at **204 tok/s**.
The DGX Spark pair generates at **65 tok/s**.
By every benchmark leaderboard metric, the 5090 is 3x faster.

But for multi-step coding tasks with thinking enabled,
the DGX pair **completes the job faster**.
And for single-turn questions,
the 5090 delivers the answer in under 2 seconds while the DGX takes 8–12 seconds.

tok/s alone told me nothing useful about actual user experience.
Here's what I learned building benchmarks for all three nodes.

---

## The Paradox

My 3-node setup:

| Node | GPU | Model | tok/s | Memory BW |
|------|-----|-------|-------|-----------|
| Desktop 5090 | RTX 5090 (32GB) | Qwen3.6-35B MoE Q4 | **204** | 1,792 GB/s |
| DGX1 | Grace Hopper (128GB) | Qwen3.5-35B FP8 | 65 | 273 GB/s |
| DGX2 | Grace Hopper (128GB) | Qwen3.6-35B FP8 | 65 | 273 GB/s |

The 5090 is 3.1x faster at generation. But:

- **For a simple factual question**: 5090 responds in 1.2s. DGX responds in 8.5s. The 5090 wins by 7x on wall-clock time.
- **For a multi-turn coding task with thinking**: 5090 completes in 45s. DGX pair (builder + reviewer) completes in 38s with higher quality.

Same model family, same quantization-adjusted quality.
The numbers invert depending on the task.

---

## Why tok/s Lies

The total time to get a response is:

```
T_response = T_prompt_processing + T_thinking + T_generation
```

tok/s only measures the last term.
Here's what it misses:

### 1. Prompt Processing (Prefill)

Before generating a single token, the model must process your entire input.
This is **memory-bandwidth bound**, not compute-bound.

| Node | Prefill Speed | 8K context prefill |
|------|--------------|-------------------|
| 5090 | ~12,000 tok/s | 0.7s |
| DGX | ~2,000 tok/s | 4.0s |

The 5090's 1,792 GB/s memory bandwidth means it chews through prompts 6x faster than the DGX's 273 GB/s.
For a long context window (32K tokens),
the DGX spends 16 seconds just reading the prompt.
The 5090 does it in 2.7 seconds.

### 2. Thinking Tokens (Invisible Cost)

With thinking enabled,
Qwen 3.6 spends **60–90% of its output tokens on internal reasoning** that never reaches the user.
If a response shows 200 visible tokens but actually generated 2,000 (with 1,800 in `<think>` blocks),
your effective speed is:

```
Effective tok/s = visible_tokens / wall_clock_time
```

On the 5090 at 204 tok/s with 90% thinking overhead:
- Raw: 204 tok/s
- Effective: **20.4 tok/s** (only 10% becomes visible output)

On the DGX at 65 tok/s with the same thinking ratio:
- Raw: 65 tok/s
- Effective: **6.5 tok/s**

The 5090 still wins in absolute terms, but the gap shrinks from 3.1x to 3.1x.
The real problem: if the DGX uses a dual-model pipeline (builder generates, reviewer validates in parallel),
the total pipeline time can beat a single-model serial workflow.

### 3. Pipeline Overhead (Multi-Step Tasks)

For coding tasks, my DGX duo runs:
1. DGX1 (Qwen 3.5) generates the code — optimized for speed, thinking OFF
2. DGX2 (Qwen 3.6) reviews and fixes — thinking ON, catches bugs

Two passes at 65 tok/s each, but with specialized roles.
The reviewer catches errors that a single-pass generator misses —
saving the retry loop that burns time on a single-GPU setup.

---

## The Framework: TTR, Effective tok/s, TCT

After measuring hundreds of tasks across all three nodes,
I defined three metrics that actually predict user satisfaction:

### TTR (Time to Response)

```
TTR = T_prefill + T_thinking + T_generation
```

The wall-clock time from pressing Enter to seeing the complete response.
This is what the user feels.

**Example — simple factual question:**

| Node | Prefill | Thinking | Generation | **TTR** |
|------|---------|----------|------------|---------|
| 5090 | 0.3s | 0s (OFF) | 0.9s | **1.2s** |
| DGX | 2.1s | 0s (OFF) | 6.4s | **8.5s** |

The 5090 is 7x faster on TTR despite being only 3.1x faster on tok/s.
The difference is prefill — memory bandwidth dominates short interactions.

### Effective tok/s

```
Effective tok/s = content_tokens / TTR
```

Only counts tokens that reach the user.
Strips thinking overhead.

| Node | Raw tok/s | Thinking overhead | **Effective tok/s** |
|------|-----------|-------------------|---------------------|
| 5090 (thinking OFF) | 204 | 0% | **204** |
| 5090 (thinking ON) | 204 | 85% | **~31** |
| DGX (thinking ON) | 65 | 85% | **~10** |

### TCT (Task Completion Time)

```
TCT = Σ(all TTR steps) + pipeline_overhead
```

For multi-step tasks (code → test → fix → verify), TCT captures the full workflow.
A faster tok/s that requires 3 retries loses to a slower tok/s that gets it right in one pass.

**Example — coding task (state machine, 3-turn):**

| Setup | Pass 1 | Pass 2 | Pass 3 | **TCT** | Quality |
|-------|--------|--------|--------|---------|---------|
| 5090 solo | 15s | 12s | 18s | **45s** | 97.1 |
| DGX duo (build+review) | 22s | 16s | — | **38s** | 100.0 |

The DGX duo finished faster *and* scored higher —
the reviewer caught the bug that would have required a third pass on the 5090.

---

## When Each Node Wins

After running 55 tasks across all three nodes (Local-Trinity benchmark suite):

| Task Type | Winner | Why |
|-----------|--------|-----|
| Single-turn factual | **5090** | Prefill speed dominates. 7x TTR advantage. |
| Simple code generation | **5090** | Thinking OFF, pure generation speed. |
| Complex multi-turn coding | **DGX duo** | Reviewer catches bugs, avoids retry loops. |
| Long-context analysis | **DGX** | 128GB VRAM, 262K context. 5090 limited to 32K. |
| Math with reasoning | **5090** | Same quality, 3x faster thinking. |

The right metric depends on the workload.
No single number captures this.

---

## Practical Implications

**If you're buying hardware for local LLM:**
- **Interactive use** (chat, quick questions): maximize memory bandwidth. Consumer GPUs win.
- **Agentic workflows** (multi-step, tool use): maximize VRAM and context window. Workstation/datacenter GPUs win.
- **Don't compare tok/s across architectures** — a 200 tok/s Q4 model on consumer GPU and a 65 tok/s FP8 model on datacenter GPU serve different purposes.

**If you're building benchmarks:**
- Report TTR for interactive tasks, TCT for agentic tasks
- Always report prefill speed separately from generation speed
- If thinking is enabled, report both raw and effective tok/s
- Never aggregate a single tok/s number across mixed workloads

---

## Data Sources

All measurements from my 3-node home lab running Qwen 3.5/3.6 35B MoE:
- **5090 harness**: 100+ scenarios, v1–v7, thinking ON/OFF A/B tests
- **DGX duo harness**: 254 tests, 14 E2E scenarios, n=3 stability verification
- **Local-Trinity**: 19 suites, 55 tasks, 290 unit tests — cross-node comparison framework
- **Latency metrics research**: [full methodology](https://github.com/ArkNill/claude-code-hidden-problem-analysis)

---

*Measured on personal hardware. Your results will vary with model size, quantization, context length, and workload mix.*
