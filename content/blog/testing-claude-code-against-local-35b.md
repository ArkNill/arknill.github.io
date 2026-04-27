---
title: "Testing Claude Code Against Local 35B Models: Building a Cross-Check Harness"
date: 2026-05-03
description: "I built three benchmark harnesses to compare Claude Code and Codex against local Qwen 35B models. The harness had more bugs than the models. Here's the v1→v7 journey — 55 tasks, 290 tests, and what 'ALL_FAIL 7→0' taught me about evaluation."
tags: ["llm", "benchmarks", "claude-code", "local-llm", "testing"]
showToc: true
TocOpen: false
draft: true
---

I run Claude Code (Opus 4.6) as my primary coding tool and pay $200/month for it.
I also run Qwen 3.5/3.6 35B locally on two DGX Sparks and an RTX 5090.
Natural question: **how does a local 35B model compare to the commercial tool I'm paying for?**

To find out, I built three separate benchmark harnesses over 10 days.
The journey taught me more about evaluation methodology than about the models themselves — because **the harness had more bugs than the models did**.

---

## The Three Harnesses

| Harness | Nodes | Focus | Tests |
|---------|-------|-------|-------|
| **cc-crosscheck** | DGX1 (3.5) | CC vs Codex vs Local — same task, three tools | 254 |
| **dgx-duo** | DGX1 + DGX2 | Builder (3.5) + Reviewer (3.6) pair mode | 254 |
| **Local-Trinity** | All 3 nodes | Unified cross-node comparison | 290 |

All three are zero-dependency Python (stdlib + urllib only).
No frameworks, no pip installs on the inference nodes.

---

## cc-crosscheck: The 3-Peer Protocol

The core idea: run the same coding task on three "peers" — Claude Code (Opus 4.6),
Codex (GPT-5.3), and a local 35B model — then compare outputs.

```
Task: "Implement a thread-safe LRU cache with TTL expiration"
     │
     ├── Claude Code (Opus 4.6, API) ──→ solution_cc.py
     ├── Codex (GPT-5.3, API) ──→ solution_codex.py
     └── Local (Qwen 3.5, DGX1) ──→ solution_local.py
     │
     ▼
  Validator: AST analysis + test execution + consensus scoring
```

The validator doesn't just check "does it run" — it performs AST consensus analysis,
comparing structural patterns across all three solutions.
If two out of three agree on an approach and one diverges, that divergence gets flagged.

### What I Found (v0.9.6, 8 scenarios)

| Scenario | CC (Opus 4.6) | Local (Qwen 3.5) | Notes |
|----------|---------------|-------------------|-------|
| LRU Cache | PASS | PASS | Structural consensus |
| Plugin Registry | PASS | PASS | — |
| Code Review | PASS | PASS | — |
| Race Condition Fix | PASS | PASS | — |
| Dijkstra | PASS | PASS | — |
| Data Pipeline | PASS | PASS | — |
| Config Merger | PASS | PASS (98.7) | Minor: nested validation edge case |
| Cache Refactor | PASS | PASS | — |

**Result: 24/24 ALL PASSED (n=3), mean score 99.6.**

The local 35B matched Claude Code quality on all 8 coding scenarios.
The gap isn't in single-task quality — it's in speed (CC responds faster due to optimized infrastructure) and in handling ambiguous,
multi-step tasks where the commercial tool's longer context and tool-use integration matter.

---

## dgx-duo: When Two Models Beat One

Instead of comparing against commercial tools,
dgx-duo asks: **can two local models working together outperform one?**

The protocol:
1. **DGX1 (Qwen 3.5, thinking OFF)** — Builder. Generates code fast.
2. **DGX2 (Qwen 3.6, thinking ON)** — Reviewer. Reads the code, finds bugs, suggests fixes.
3. If the reviewer finds issues → builder gets a second pass with feedback.

### Results (v0.9.6, 14 E2E scenarios)

| Mode | Scenarios | Pass Rate | Mean Score |
|------|-----------|-----------|------------|
| Single (DGX1 alone) | 8 | 75% | 89.2 |
| **Pair (builder + reviewer)** | 8 | **100%** | **99.6** |
| Multi-step | 6 | 83% | 96.1 |

The reviewer catches bugs that a single-pass generator misses.
The most common catches:
- Missing error handling (pytest.raises coverage)
- Edge cases in concurrent code (race conditions)
- Incomplete interface implementations

### The Rubber Stamp Problem (v0.9.3→v0.9.4)

Early versions had a critical flaw: the reviewer **always approved**.
It would say "looks good" even when the code had obvious bugs.
This isn't a model limitation — it's a prompt engineering failure.

The fix (v0.9.4): changed the reviewer from "verify this code works" to "review this code — you are empowered to fail it."
Giving the model explicit authority to reject transformed its behavior.
Rubber stamp rate dropped from >90% to <10%.

---

## Local-Trinity: The Unified Benchmark

Local-Trinity combines all three nodes into a single benchmark suite.
Same task, all three nodes, n=3 for stability:

### Architecture

```
Local-Trinity (ZBook orchestrator)
├── 17 source modules, 5,850 LOC
├── 290 unit tests
├── 19 suites / 55 tasks
│
├── Node: DGX1 (Qwen 3.5 FP8, 65 tok/s)
├── Node: DGX2 (Qwen 3.6 FP8, 65 tok/s)
└── Node: Desktop5090 (Qwen 3.6 Q4, 204 tok/s)
```

### Task Categories

| Category | Suites | Tasks | Examples |
|----------|--------|-------|----------|
| Factual (KO/EN) | 5 | 16 | Korean trivia, English facts, ambiguous questions |
| Math | 2 | 7 | Arithmetic, modular, combinatorics |
| Coding | 6 | 14 | Single-file, debug, multi-turn, algorithms |
| Hard | 4 | 12 | 3-turn state machines, B-Trees, regex engines |
| Agent | 2 | 6 | Speed-focused, algorithm-heavy |

### The v1→v7 Journey

This is where the real learning happened:

| Version | ALL_FAIL | What Broke | Root Cause |
|---------|----------|-----------|------------|
| **v1** | **7** | Models "failing" tasks they could clearly solve | Harness bugs: wrong assertions, broken scoring |
| v2 | 1 | Still one persistent failure | Difficulty too high for Q4 |
| v3 | 2 | Regression from "improvements" | AST scoring introduced new bugs |
| v4 | 2 | Same two won't go away | try_pass counting was wrong |
| **v5** | **0** | Finally all passing | atexit counting, prompt rewording |
| v6 | 0 | Stable — added features | Performance timing, thinking A/B |
| **v7** | **0** | Stable — final polish | assert_total denominator fix |

**The harness had more bugs than the models.**
Every time I thought a model was failing, the actual problem was:
- Assert matching that rejected valid alternative formats
- Scoring that penalized correct-but-different approaches
- Timeouts too short for thinking-heavy tasks
- Prompts that were ambiguous to the model but clear to me

### v5 Benchmark Results (12 Hard Tasks, n=3)

| Task | 5090 (Q4) | DGX1 (3.5 FP8) | DGX2 (3.6 FP8) | Verdict |
|------|-----------|-----------------|-----------------|---------|
| EventEmitter | 98.8 | 97.9 | 97.7 | UNANIMOUS |
| StateMachine | 97.1 | 100.0 | 97.1 | UNANIMOUS |
| Mini ORM | 100.0 | 70.6 | 100.0 | MAJORITY |
| TaskScheduler | 100.0 | 76.5 | 100.0 | MAJORITY |
| ExprParser | 100.0 | 99.5 | 99.9 | UNANIMOUS |
| KV Store | 98.0 | 97.6 | 98.0 | UNANIMOUS |
| System Design | 100.0 | 97.7 | 97.4 | UNANIMOUS |
| Combinatorics | 100.0 | 100.0 | 99.9 | UNANIMOUS |
| Code Review | 80.0 | 99.2 | 79.1 | MAJORITY |
| Regex Engine | 99.6 | 97.3 | 98.3 | UNANIMOUS |
| JSON Parser | 98.5 | 97.9 | 98.1 | UNANIMOUS |
| B-Tree | 97.8 | 39.0 | 97.7 | MAJORITY |

**UNANIMOUS 8 / MAJORITY 4 / ALL_FAIL 0**

The MAJORITY cases reveal real model differences:
- **DGX1 (Qwen 3.5)** struggles with multi-turn tasks requiring state tracking
- **B-Tree**: DGX1 consistently fails on node splitting (structural limitation)
- **Code Review**: 5090 and DGX2 both miss pickle security check (non-deterministic)

---

## What I Learned

### 1. Benchmark Quality > Model Quality

The models were fine from v1.
My evaluation framework was broken.
If your benchmark shows a model "failing" a task it should clearly handle, **the bug is in your benchmark**.

### 2. Scores ≠ Quality

A score of 100.0 means "passed all structural checks" — not "produced good code."
I found cases where code scored 100.0 but had subtle bugs (nested validation edge cases) that the validator couldn't catch.
The validator measures what it measures, not what you think it measures.

### 3. n=3 Is Mandatory

With n=1, you can't distinguish model limitation from inference non-determinism.
vLLM's MTP causes one run in four to produce wildly different output.
n=3 with mean scoring separates signal from noise.

### 4. The 35B Sweet Spot

For coding tasks with clear specifications,
Qwen 35B MoE (3B active) at FP8 matches commercial tools on quality.
The gap appears in:
- Ambiguous multi-step tasks (commercial tools have better instruction following)
- Long-context retrieval (128GB vs 1M tokens cloud-side)
- Tool use and file system interaction (Claude Code's integration layer)

For everything else — writing functions, debugging, algorithms, refactoring —
the local model works.

---

## Code

All three harnesses are zero-dependency Python:
- **cc-crosscheck**: `~/GitHub/cc-crosscheck/` — 254 tests, dual-35B architecture
- **dgx-duo**: `~/GitHub/dgx-duo/` — 9 modules, 254 tests, pair mode
- **Local-Trinity**: `~/GitHub/local-trinity/` — 17 modules, 290 tests, 3-node unified

Public release planned after documentation pass.

---

*Personal benchmarks on personal hardware.
Models: Qwen 3.5/3.6 35B-A3B (FP8 and Q4).
Commercial comparison: Claude Code with Opus 4.6 (v2.1.109).
Not affiliated with Anthropic, OpenAI, or Alibaba.*
