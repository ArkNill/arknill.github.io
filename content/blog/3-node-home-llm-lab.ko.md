---
title: "3노드 홈 LLM 랩을 구축했습니다. 실제로 무엇이 필요한지 알려드립니다."
date: 2026-04-27
description: "DGX Spark 2대(각 128GB)와 RTX 5090 데스크톱 — Qwen 3.5/3.6 35B를 프로덕션으로 운영합니다. 하드웨어 선택, 실제 비용, 운영 교훈, 그리고 왜 3노드가 하나의 큰 머신보다 나은지 이야기합니다."
tags: ["llm", "local-llm", "hardware", "dgx", "homelab"]
showToc: true
TocOpen: false
draft: false
---

저는 집에서 3노드 로컬 LLM 추론 클러스터를 운영합니다.
NVIDIA DGX Spark 2대(각 128GB 통합 메모리)와 RTX 5090 데스크톱 1대(32GB VRAM)로 구성되어 있습니다.
세 노드 모두 Qwen 3.5/3.6 35B MoE 모델을 로컬 네트워크에서 24/7 서빙합니다.

이것은 주말 실험이 아닙니다 — 매일 사용하는 개발 인프라입니다.
모든 코드 리뷰, 모든 리서치 쿼리, 모든 벤치마크가 이 노드들을 대상으로 실행됩니다.
셋업의 구성, 비용, 그리고 스펙 시트에는 없는 운영 교훈을 공유합니다.

---

## 하드웨어

| 노드 | CPU | GPU/메모리 | 스토리지 | 역할 |
|------|-----|-----------|---------|------|
| **DGX Spark #1** | NVIDIA Grace (ARM, 20C) | GB10, 128GB LPDDR5X UMA | 932GB NVMe | 코딩 — Qwen 3.5 FP8, vLLM |
| **DGX Spark #2** | NVIDIA Grace (ARM, 20C) | GB10, 128GB LPDDR5X UMA | 932GB NVMe | 에이전틱 — Qwen 3.6 FP8, vLLM |
| **Desktop 5090** | (Windows 호스트) | RTX 5090, 32GB GDDR7 | — | 인터랙티브 — Qwen 3.6 MoE Q4, llama.cpp |
| **ZBook Ultra** | Ryzen AI MAX+ PRO 395 | Radeon 8060S (내장) | 2TB NVMe | 오케스트레이터 — SSH 허브, Claude Code, 개발 환경 |

모든 노드는 동일 LAN(192.168.0.x)에 기가비트 유선으로 연결되어 있습니다. ZBook이 SSH를 통해 모든 것을 오케스트레이션합니다.

### 왜 하나의 큰 머신 대신 3노드인가

단일 128GB 노드로도 모델을 실행할 수 있습니다.
하지만 **역할 특화**가 클러스터를 부분의 합보다 강하게 만듭니다:

- **DGX1 (Qwen 3.5, thinking OFF):** 순수 코드 생성. 65 tok/s, 빠른 빌더 출력에 최적화. thinking 오버헤드 없음.
- **DGX2 (Qwen 3.6, thinking ON):** 코드 리뷰와 에이전틱 태스크. 빌더가 놓친 버그를 잡습니다. 깊은 분석을 위해 thinking 활성화.
- **5090 (Q4, 204 tok/s):** 인터랙티브 쿼리. 사실 확인 질문에 2초 미만 응답. 품질(Q4 vs FP8)을 속도 3배로 교환.

함께 작동할 때(DGX1이 빌드, DGX2가 리뷰),
이 페어는 **24개 시나리오에서 99.6/100** — 어느 한 노드 단독보다 높은 점수를 기록합니다.

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

두 DGX 노드 모두 자동 재시작이 설정된 systemd 서비스로 실행됩니다.
GPU 클럭은 2500 MHz로 고정되어 있습니다(`dgx-gpu-clocks.service`).
DGX2는 누적된 KV cache 단편화를 해소하기 위해 매일 05:00 KST에 vLLM을 재시작하는 타이머가 있습니다.

### Desktop 5090: llama.cpp

```
llama-server (llama.cpp, sm_120 build)
├── Model: Qwen3.6-35B-MoE-Q4_K_XL (23.1GB)
├── Context: 32K tokens
├── Speed: 204 tok/s generation, ~12000 tok/s prefill
├── Thinking: Per-request dynamic control
└── Port: 8080 (OpenAI-compatible API)
```

5090의 1,792 GB/s 메모리 대역폭은 압도적인 prefill 속도를 제공합니다 — DGX보다 6배 빠릅니다.
단일 턴 쿼리의 경우 2초 이내에 응답합니다.

### ZBook: 오케스트레이션

ZBook은 모델을 실행하지 않습니다 — 그 외 모든 것을 실행합니다:
- Claude Code (주 개발 환경)
- 모든 노드에 대한 SSH 관리
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

모두 기가비트 유선입니다.
측정된 노드 간 처리량: 674–936 Mbps.
ZBook과 각 노드 간 API 호출: 2ms 미만 지연.

DGX Spark는 ARM(aarch64)입니다 — Docker 이미지와 컴파일된 의존성에서 이 점이 중요합니다.
vLLM의 공식 ARM 이미지는 동작하지만, 커스텀 도구는 ARM 빌드가 필요합니다.

---

## 운영 현실

### 전력과 발열

- DGX Spark: 부하 시 각 ~150W. 조용합니다(서버급 팬, 거의 들리지 않음).
- Desktop 5090: 풀 GPU 부하 시 ~400W. 지속적 추론 시 소음이 큽니다.
- ZBook: 45–65W. 대부분의 작업에서 팬리스.

클러스터 총합: 피크 ~750W, 통상 ~400W.
전기 요금이 월 약 $40 증가했습니다.

### 고장 나는 것들

**vLLM 메모리 누수.** 3–5일간 연속 추론 후
KV cache 단편화가 처리량을 10–15% 저하시킵니다.
DGX2의 일일 재시작 타이머가 이 문제를 해결했습니다.
DGX1은 더 안정적입니다(thinking 집약적 요청이 적음).

**MTP 비결정성.** 같은 프롬프트, 같은 모델, 다른 출력.
vLLM의 speculative decoding은 llama.cpp에는 없는 분산을 도입합니다.
벤치마크에서는 n=3으로 실행하고 평균을 취합니다.
프로덕션 코드 생성의 경우, 5090의 단일 샷 결과만 신뢰합니다.

**ARM에서의 Docker.** 대부분의 커뮤니티 Docker 이미지는 x86 전용입니다.
ARM에서 vLLM을 소스 빌드하는 데 45분이 걸렸습니다.
일부 Python wheel은 컴파일이 필요합니다.
공식 NVIDIA DGX 이미지는 이를 처리하지만, 커스텀 작업에는 인내심이 필요합니다.

**네트워크 중단.** 라우터의 짧은 전원 순간 단절이 모든 SSH 세션을 죽입니다.
중요한 모든 것에 tmux/nohup 대신 systemd 서비스를 사용합니다.
모델은 부팅 시 자동 재시작됩니다.

### 모델 업데이트

DGX에서 모델을 교체하는 데 약 30분이 걸립니다:
1. HuggingFace에서 새 모델 다운로드 (~35GB for 35B FP8)
2. vLLM 실행 스크립트 업데이트
3. `systemctl restart dgx-vllm`
4. 검증 스위트 실행 (5분)

이 방법으로 DGX2를 Qwen 3.5에서 3.6으로 전환했습니다.
이전 모델은 백업으로 디스크에 유지합니다 — 932GB는 넉넉합니다.

---

## 이 인프라가 가능하게 하는 것

3노드로 다음이 가능합니다:

1. **상용 AI 도구 크로스체크** — 같은 코딩 태스크를 Claude Code, Codex, 로컬 35B에서 실행합니다. 단일 벤더를 신뢰하지 않고 품질을 비교합니다.
2. **모델 버전 A/B 테스트** — DGX1은 3.5, DGX2는 3.6을 실행합니다. 같은 인프라에서 통제된 비교가 가능합니다.
3. **페어 프로그래밍 파이프라인 실행** — Builder(빠르게, thinking 없이) → Reviewer(느리지만, 버그 포착). 24개 시나리오에서 99.6/100.
4. **모든 것을 벤치마크** — 55개 태스크, 290개 유닛 테스트, 19개 스위트. 모든 노드에서 재현 가능한 측정.
5. **독립성 유지** — Claude Code에 할당량 문제나 모델 회귀가 발생해도 작업이 멈추지 않습니다.

---

## 솔직히 가치가 있습니까?

**경제성을 평가한다면:** Claude Code Max 20은 월 $200입니다.
DGX Spark의 초기 비용은 상당히 더 높습니다.
순수 토큰당 비용으로 보면, 클라우드가 몇 년간 이깁니다.

**역량을 평가한다면:** 어떤 클라우드 API도 이 수준의 통제권을 제공하지 않습니다 —
요청별 thinking 토글, 모델 고정, 결정적 추론,
크로스 모델 파이프라인, 제로 레이턴시 로컬 접근, 완전한 프라이버시.

**학습을 평가한다면:** 이것을 구축하고 운영하면서 어떤 논문이나 벤치마크 리더보드보다 LLM 추론에 대해 더 많이 배웠습니다.
실패 모드(양자화 손실, 비결정성, thinking 폭주)는 공개된 벤치마크에 나타나지 않습니다.
수천 개의 실제 태스크를 실행해야만 발견할 수 있습니다.

이 클러스터는 클라우드보다 저렴하지 않습니다.
제 작업에 중요한 특정 방식에서 **더 높은 역량**을 제공합니다 —
그리고 벤더가 나쁜 업데이트를 배포해도 성능이 저하되지 않습니다.

---

## 스펙 요약

| 구성요소 | DGX Spark (×2) | Desktop 5090 | ZBook Ultra |
|-----------|---------------|-------------|-------------|
| 메모리 | 128GB UMA | 32GB GDDR7 | 108GB UMA |
| 대역폭 | 273 GB/s | 1,792 GB/s | — |
| 최대 컨텍스트 | 262K | 32K | — |
| 생성 속도 | 65 tok/s (FP8) | 204 tok/s (Q4) | — |
| 정밀도 | FP8 (production) | Q4 (interactive) | — |
| 백엔드 | vLLM + MTP | llama.cpp | — |
| 전력 | ~150W | ~400W | ~50W |
| 역할 | 프로덕션 추론 | 인터랙티브 / 속도 | 오케스트레이션 |

---

*개인 인프라입니다. 하드웨어 가용성과 가격은 지역에 따라 다릅니다.*
