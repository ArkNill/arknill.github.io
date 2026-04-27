---
title: "3노드 홈 LLM 랩을 만들었습니다. 실제로 필요한 것들."
date: 2026-04-26
description: "DGX Spark 2대(128GB)와 RTX 5090 데스크탑 — Qwen 3.5/3.6 35B를 프로덕션으로 운영합니다. 하드웨어 선택, 실제 비용, 운영 교훈, 그리고 왜 3대가 1대보다 나은지."
tags: ["llm", "local-llm", "hardware", "dgx", "homelab"]
showToc: true
TocOpen: false
draft: false
---

집에서 3노드 LLM 추론 클러스터를 운영하고 있습니다.
NVIDIA DGX Spark 2대(각 128GB 통합 메모리)와 RTX 5090 데스크탑(32GB VRAM) 1대.
세 노드 모두 Qwen 3.5/3.6 35B MoE 모델을 24/7 로컬 네트워크에서 서빙합니다.

주말 실험이 아닙니다.
매일 쓰는 개발 인프라입니다.
코드 리뷰, 리서치 쿼리, 벤치마크 — 전부 이 노드들 위에서 돌아갑니다.

---

## 왜 3대가 필요한가

"128GB면 모델 하나 올리기 충분하지 않나요?"

맞습니다. 1대로 모델은 돌아갑니다.
그런데 **역할 분리(role specialization)**를 하면 클러스터가 단일 노드의 합보다 강해집니다.

- **DGX1 (Qwen 3.5, thinking OFF):** 순수 코드 생성 전용. 65 tok/s, thinking 오버헤드 없이 빠른 빌더 출력.
- **DGX2 (Qwen 3.6, thinking ON):** 코드 리뷰와 에이전틱 태스크. 빌더가 놓친 버그를 잡습니다. 깊은 분석을 위해 thinking 활성화.
- **5090 (Q4, 204 tok/s):** 인터랙티브 쿼리. 2초 이내 응답. 품질(Q4 vs FP8)을 속도 3배와 교환.

DGX1이 코드를 짜고 DGX2가 리뷰하는 파이프라인을 돌리면,
**24개 시나리오에서 99.6/100** — 어느 한 노드 단독보다 높습니다.

1대로는 빌드와 리뷰를 동시에 할 수 없습니다.
2대로는 인터랙티브 속도가 부족합니다.
3대여야 역할이 겹치지 않습니다.

---

## 하드웨어 구성

| 노드 | CPU | GPU/메모리 | 스토리지 | 역할 |
|------|-----|-----------|---------|------|
| **DGX Spark #1** | NVIDIA Grace (ARM, 20C) | GB10, 128GB LPDDR5X UMA | 932GB NVMe | Coding — Qwen 3.5 FP8, vLLM |
| **DGX Spark #2** | NVIDIA Grace (ARM, 20C) | GB10, 128GB LPDDR5X UMA | 932GB NVMe | Agentic — Qwen 3.6 FP8, vLLM |
| **Desktop 5090** | (Windows host) | RTX 5090, 32GB GDDR7 | — | Interactive — Qwen 3.6 MoE Q4, llama.cpp |
| **ZBook Ultra** | Ryzen AI MAX+ PRO 395 | Radeon 8060S (integrated) | 2TB NVMe | Orchestrator — SSH hub, Claude Code, dev env |

전체 노드가 동일 LAN (192.168.0.x)에 유선 기가비트로 연결되어 있습니다.
ZBook이 SSH로 모든 노드를 오케스트레이션합니다.

---

## 소프트웨어 스택

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

두 DGX 모두 systemd 서비스로 자동 재시작 설정되어 있습니다.
GPU 클럭 2500 MHz 고정 (`dgx-gpu-clocks.service`).
DGX2는 매일 05:00 KST에 vLLM 재시작 타이머가 동작합니다 — KV cache fragmentation 누적을 해소하기 위한 조치입니다.

### Desktop 5090: llama.cpp

```
llama-server (llama.cpp, sm_120 build)
├── Model: Qwen3.6-35B-MoE-Q4_K_XL (23.1GB)
├── Context: 32K tokens
├── Speed: 204 tok/s generation, ~12000 tok/s prefill
├── Thinking: Per-request dynamic control
└── Port: 8080 (OpenAI-compatible API)
```

5090의 메모리 대역폭 1,792 GB/s가 prefill 속도를 압도적으로 끌어올립니다 — DGX 대비 6배.
싱글 턴 쿼리는 2초 이내에 응답이 돌아옵니다.

### ZBook: 오케스트레이션

ZBook은 모델을 돌리지 않습니다. 나머지 전부를 돌립니다:
- Claude Code (메인 개발 환경)
- 전 노드 SSH 관리
- 벤치마크 하네스 (Local-Trinity, dgx-duo, 5090-harness)
- Docker 서비스: llm-relay proxy, Neo4j, Redis, Qdrant, SearXNG

---

## 네트워크 토폴로지

```
[Internet] ← [Router .1]
                ├── Desktop5090  .16 (Windows, llama-server)
                ├── DGX1         .26 (Ubuntu ARM, vLLM)
                ├── DGX2         .31 (Ubuntu ARM, vLLM)
                └── [Ethernet Hub]
                     └── ZBook   .34 (Ubuntu, orchestrator)
```

전부 유선 기가비트입니다.
실측 노드 간 스루풋: 674–936 Mbps.
ZBook에서 각 노드까지 API 레이턴시: <2ms.

DGX Spark는 ARM (aarch64)입니다.
Docker 이미지와 컴파일 의존성 선택에 직접적으로 영향을 줍니다.
vLLM 공식 ARM 이미지는 잘 동작하지만, 커스텀 툴링은 ARM 빌드가 필요합니다.

---

## 운영 현실

### 전력과 발열

- DGX Spark: 각 ~150W (부하 시). 조용합니다 — 서버급 팬인데 거의 들리지 않습니다.
- Desktop 5090: ~400W (풀 GPU 부하). 지속 추론 시 팬 소음이 있습니다.
- ZBook: 45–65W. 대부분의 작업에서 팬리스.

전체 클러스터: 피크 ~750W, 일상 ~400W.
전기요금 증가분은 월 약 $40입니다.
한국 주택용 전기요금(누진 2구간 기준)으로 환산하면 월 4–5만원 수준입니다.

### 고장 나는 것들

**vLLM 메모리 누수.**
3–5일 연속 추론 후 KV cache fragmentation이 쌓이면서 스루풋이 10–15% 떨어집니다.
DGX2의 일일 재시작 타이머로 해결했습니다.
DGX1은 비교적 안정적입니다 — thinking이 무거운 요청이 적기 때문입니다.

**MTP 비결정성(non-determinism).**
같은 프롬프트, 같은 모델인데 출력이 다릅니다.
vLLM의 speculative decoding이 분산을 만들어냅니다 — llama.cpp에는 없는 문제입니다.
벤치마크에서는 n=3으로 돌려서 평균을 냅니다.
프로덕션 코드 생성은 5090의 단일 샷 결과만 신뢰합니다.

**ARM에서의 Docker.**
커뮤니티 Docker 이미지 대부분이 x86 전용입니다.
ARM에서 vLLM 소스 빌드에 45분이 걸렸습니다.
일부 Python wheel은 네이티브 컴파일이 필요합니다.
NVIDIA 공식 DGX 이미지는 처리되어 있지만, 커스텀 작업에는 인내심이 필요합니다.

**네트워크 중단.**
라우터에 순간 정전이 오면 전체 SSH 세션이 한꺼번에 끊어집니다.
중요 서비스는 전부 systemd로 운영합니다 (tmux/nohup이 아닙니다).
모델은 부팅 시 자동 재시작됩니다.

### 모델 교체

DGX에서 모델 교체는 약 30분이면 됩니다:
1. HuggingFace에서 새 모델 다운로드 (35B FP8 기준 ~35GB)
2. vLLM 시작 스크립트 업데이트
3. `systemctl restart dgx-vllm`
4. 검증 스위트 실행 (5분)

이 방식으로 DGX2를 Qwen 3.5에서 3.6으로 전환했습니다.
이전 모델은 백업으로 디스크에 남겨둡니다 — 932GB면 넉넉합니다.

---

## 이 구성으로 할 수 있는 것

3노드가 있으면:

1. **상용 AI 도구 크로스체크** — 같은 코딩 태스크를 Claude Code, Codex, 로컬 35B에서 실행합니다. 단일 벤더를 맹신하지 않고 품질 비교가 가능합니다.
2. **모델 버전 A/B 테스트** — DGX1은 3.5, DGX2는 3.6. 동일 인프라에서 통제된 비교.
3. **페어 프로그래밍 파이프라인** — Builder (빠르게, thinking 없이) → Reviewer (느리지만 버그 포착). 24 시나리오에서 99.6/100.
4. **모든 것을 벤치마크** — 55 태스크, 290 단위 테스트, 19 스위트. 전 노드에서 재현 가능한 측정.
5. **독립성 유지** — Claude Code에 쿼터 이슈가 생기거나 모델 리그레션이 발생해도 작업이 멈추지 않습니다.

---

## 그래서 가치가 있습니까?

**비용으로 평가하면:**
Claude Code Max 20이 월 $200입니다.
DGX Spark 2대의 초기 비용은 그보다 훨씬 큽니다.
순수 토큰당 비용으로 따지면, 클라우드가 수 년간 이깁니다.

**역량으로 평가하면:**
어떤 클라우드 API도 이 수준의 제어를 주지 않습니다 —
요청별 thinking 토글, 모델 고정, 결정론적 추론,
크로스 모델 파이프라인, 제로 레이턴시 로컬 접근, 완전한 프라이버시.

**학습으로 평가하면:**
이 클러스터를 구축하고 운영하면서 어떤 논문이나 벤치마크 리더보드보다 LLM 추론에 대해 더 많이 배웠습니다.
장애 모드(양자화 손실, 비결정성, thinking 폭주)는 공개된 벤치마크에 나타나지 않습니다.
수천 개의 실제 태스크를 돌려봐야만 발견됩니다.

이 클러스터는 클라우드보다 싸지 않습니다.
제 작업에 중요한 특정 측면에서 **더 높은 역량**을 제공합니다 —
그리고 벤더가 나쁜 업데이트를 배포해도 영향을 받지 않습니다.

---

## 스펙 요약

| 항목 | DGX Spark (×2) | Desktop 5090 | ZBook Ultra |
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

*개인 인프라입니다. 하드웨어 가용성과 가격은 지역에 따라 다릅니다.*
