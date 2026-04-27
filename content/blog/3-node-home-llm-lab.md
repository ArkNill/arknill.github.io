---
title: "I Built a 3-Node Home LLM Lab. Here's What It Actually Takes."
date: 2026-04-27
description: "Two DGX Sparks (128GB each) and an RTX 5090 desktop — running Qwen 3.5/3.6 35B in production. Hardware choices, real costs, operational lessons, and why three nodes beat one big one."
tags: ["llm", "local-llm", "hardware", "dgx", "homelab"]
showToc: true
TocOpen: false
draft: false
---

I run a 3-node local LLM inference cluster at home.
Two NVIDIA DGX Sparks (128GB unified memory each) and one RTX 5090 desktop (32GB VRAM).
All three serve Qwen 3.5/3.6 35B MoE models 24/7 over my local network.

This isn't a weekend experiment — it's my daily development infrastructure.
Every code review, every research query, every benchmark runs against these nodes.
Here's what the setup looks like, what it costs, and what I learned that no spec sheet tells you.

---

## The Hardware

| Node | CPU | GPU/Memory | Storage | Role |
|------|-----|-----------|---------|------|
| **DGX Spark #1** | NVIDIA Grace (ARM, 20C) | GB10, 128GB LPDDR5X UMA | 932GB NVMe | Coding — Qwen 3.5 FP8, vLLM |
| **DGX Spark #2** | NVIDIA Grace (ARM, 20C) | GB10, 128GB LPDDR5X UMA | 932GB NVMe | Agentic — Qwen 3.6 FP8, vLLM |
| **Desktop 5090** | (Windows host) | RTX 5090, 32GB GDDR7 | — | Interactive — Qwen 3.6 MoE Q4, llama.cpp |
| **ZBook Ultra** | Ryzen AI MAX+ PRO 395 | Radeon 8060S (integrated) | 2TB NVMe | Orchestrator — SSH hub, Claude Code, dev env |

All nodes on the same LAN (192.168.0.x), gigabit wired. The ZBook orchestrates everything via SSH.

### Why Three Nodes Instead of One Big Machine

A single 128GB node can run the model,
but **role specialization** makes the cluster stronger than the sum of its parts:

- **DGX1 (Qwen 3.5, thinking OFF):** Pure code generation. 65 tok/s, optimized for fast builder output. No thinking overhead.
- **DGX2 (Qwen 3.6, thinking ON):** Code review and agentic tasks. Catches bugs the builder missed. Thinking enabled for deeper analysis.
- **5090 (Q4, 204 tok/s):** Interactive queries. Sub-2-second responses for factual questions. Trading quality (Q4 vs FP8) for 3x speed.

When they work together (DGX1 builds, DGX2 reviews),
the pair scores **99.6/100 across 24 scenarios** — higher than either node alone.

---

## The Software Stack

### DGX Spark: vLLM + MTP

```
vLLM 0.19.1 (Docker: vllm-node:latest, 18.2GB image)
├── Model: Qwen3.5-35B-A3B-FP8 (DGX1) / Qwen3.6-35B-A3B-FP8 (DGX2)
├── Context: 262K tokens (FP8 KV cache)
├── Speculative: MTP (Multi-Token Prediction), depth=3
├── Backend: FlashInfer
├── Port: 30000 (OpenAI-compatible API)
└── Speed: 65 tok/s generation, ~2000 tok/s prefill
```

Both DGX nodes run as systemd services with automatic restart.
GPU clocks locked at 2500 MHz (`dgx-gpu-clocks.service`).
DGX2 has a daily vLLM restart timer at 05:00 KST to clear accumulated KV cache fragmentation.

### Desktop 5090: llama.cpp

```
llama-server (llama.cpp, sm_120 build)
├── Model: Qwen3.6-35B-MoE-Q4_K_XL (23.1GB)
├── Context: 32K tokens
├── Speed: 204 tok/s generation, ~12000 tok/s prefill
├── Thinking: Per-request dynamic control
└── Port: 8080 (OpenAI-compatible API)
```

The 5090's 1,792 GB/s memory bandwidth gives it absurd prefill speed — 6x faster than the DGX.
For single-turn queries, it responds in under 2 seconds.

### ZBook: Orchestration

The ZBook doesn't run models — it runs everything else:
- Claude Code (primary dev environment)
- SSH management to all nodes
- Benchmark harnesses (Local-Trinity, dgx-duo, 5090-harness)
- Docker services: llm-relay proxy, Neo4j, Redis, Qdrant, SearXNG

---

## Network Topology

```
[Internet] ← [Router .1]
                ├── Desktop5090  .16 (Windows, llama-server)
                ├── DGX1         .26 (Ubuntu ARM, vLLM)
                ├── DGX2         .31 (Ubuntu ARM, vLLM)
                └── [Ethernet Hub]
                     └── ZBook   .34 (Ubuntu, orchestrator)
```

All wired gigabit.
Measured inter-node throughput: 674–936 Mbps.
API calls between ZBook and any node: <2ms latency.

The DGX Sparks are ARM (aarch64) — this matters for Docker images and compiled dependencies.
vLLM's official ARM images work, but custom tooling needs ARM builds.

---

## Operational Realities

### Power and Heat

- DGX Spark: ~150W each under load. Quiet (server-grade fans, rarely audible).
- Desktop 5090: ~400W under full GPU load. Loud under sustained inference.
- ZBook: 45–65W. Fanless for most tasks.

Total cluster: ~750W peak, ~400W typical.
My electricity bill increased by about $40/month.

### What Breaks

**vLLM memory leaks.** After 3–5 days of continuous inference,
KV cache fragmentation degrades throughput by 10–15%.
The daily restart timer on DGX2 solved this.
DGX1 is more stable (fewer thinking-heavy requests).

**MTP non-determinism.** Same prompt, same model, different output.
vLLM's speculative decoding introduces variance that llama.cpp doesn't have.
I run n=3 and take the mean for benchmarks.
For production code generation, I only trust single-shot results from the 5090.

**Docker on ARM.** Most community Docker images are x86-only.
Building vLLM from source on ARM took 45 minutes.
Some Python wheels need compilation.
The official NVIDIA DGX images handle this, but anything custom requires patience.

**Network interruptions.** A brief power blink at the router kills all SSH sessions.
I use systemd services (not tmux/nohup) for everything critical.
Models auto-restart on boot.

### Model Updates

Switching models on a DGX takes about 30 minutes:
1. Download new model from HuggingFace (~35GB for 35B FP8)
2. Update the vLLM launch script
3. `systemctl restart dgx-vllm`
4. Run validation suite (5 minutes)

I've switched DGX2 from Qwen 3.5 to 3.6 this way.
The old model stays on disk as backup — 932GB is generous.

---

## What This Enables

With three nodes, I can:

1. **Cross-check commercial AI tools** — Run the same coding task on Claude Code, Codex, and local 35B. Compare quality without trusting any single vendor.
2. **A/B test model versions** — DGX1 runs 3.5, DGX2 runs 3.6. Same infrastructure, controlled comparison.
3. **Run pair programming pipelines** — Builder (fast, no thinking) → Reviewer (slower, catches bugs). 99.6/100 on 24 scenarios.
4. **Benchmark everything** — 55 tasks, 290 unit tests, 19 suites. Reproducible measurements across all nodes.
5. **Stay independent** — When Claude Code has quota issues or model regressions, my work doesn't stop.

---

## Honestly, Is It Worth It?

**If you're evaluating the economics:** Claude Code Max 20 is $200/month.
The DGX Sparks cost significantly more upfront.
Pure cost-per-token, the cloud wins for years.

**If you're evaluating the capability:** No cloud API gives you this level of control —
per-request thinking toggles, model pinning, deterministic inference,
cross-model pipelines, zero-latency local access, complete privacy.

**If you're evaluating the learning:** Building and operating this taught me more about LLM inference than any paper or benchmark leaderboard.
The failure modes (quantization loss, non-determinism, thinking runaways) don't appear in published benchmarks.
You only find them by running thousands of real tasks.

The cluster isn't cheaper than the cloud. It's **more capable** in specific ways that matter for my work — and it doesn't degrade when a vendor ships a bad update.

---

## Specs Summary

| Component | DGX Spark (×2) | Desktop 5090 | ZBook Ultra |
|-----------|---------------|-------------|-------------|
| Memory | 128GB UMA | 32GB GDDR7 | 108GB UMA |
| Bandwidth | 273 GB/s | 1,792 GB/s | — |
| Max Context | 262K | 32K | — |
| Generation | 65 tok/s (FP8) | 204 tok/s (Q4) | — |
| Precision | FP8 (production) | Q4 (interactive) | — |
| Backend | vLLM + MTP | llama.cpp | — |
| Power | ~150W | ~400W | ~50W |
| Role | Production inference | Interactive / speed | Orchestration |

---

*Personal infrastructure. Hardware availability and pricing vary by region.*
