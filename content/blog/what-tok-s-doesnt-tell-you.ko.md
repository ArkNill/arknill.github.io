---
title: "tok/s가 알려주지 않는 것: 실제로 체감되는 LLM 속도 측정법"
date: 2026-04-24
description: "204 tok/s GPU가 65 tok/s GPU보다 느리게 느껴지는 작업이 있습니다. tok/s 하나로는 실사용 체감 속도를 예측할 수 없습니다. 실측에서 도출한 3가지 프레임워크(TTR, Effective tok/s, TCT)를 소개합니다."
tags: ["llm", "benchmarks", "inference", "local-llm"]
showToc: true
TocOpen: false
draft: false
---

tok/s 숫자만 보고 GPU를 고르면 실패합니다.

저는 Qwen 3.6 35B를 3대의 머신에서 운영하고 있습니다.
RTX 5090은 **204 tok/s**를 뽑습니다.
DGX Spark 2대는 각각 **65 tok/s**입니다.
벤치마크 리더보드 기준으로 5090이 3배 빠릅니다.

그런데 thinking을 켜고 멀티스텝 코딩을 시키면,
DGX 조합이 **더 먼저 끝납니다**.
반면 단순 질문에는 5090이 2초 만에 답하고, DGX는 8~12초 걸립니다.

tok/s 하나로는 실제 사용 경험을 전혀 예측할 수 없었습니다.
3노드 벤치마크를 직접 만들면서 배운 것을 정리합니다.

---

## 역설: 3배 빠른 GPU가 더 느린 순간

제 3노드 구성입니다.

| Node | GPU | Model | tok/s | Memory BW |
|------|-----|-------|-------|-----------|
| Desktop 5090 | RTX 5090 (32GB) | Qwen3.6-35B MoE Q4 | **204** | 1,792 GB/s |
| DGX1 | Grace Hopper (128GB) | Qwen3.5-35B FP8 | 65 | 273 GB/s |
| DGX2 | Grace Hopper (128GB) | Qwen3.6-35B FP8 | 65 | 273 GB/s |

5090의 생성 속도는 DGX의 3.1배입니다. 하지만 실제 작업에서는 이렇게 됩니다.

- **간단한 팩트 질문**: 5090은 1.2초, DGX는 8.5초. 5090이 월클 기준 7배 빠릅니다.
- **thinking ON + 멀티턴 코딩**: 5090은 45초, DGX duo(빌더+리뷰어)는 38초에 더 높은 품질로 완료됩니다.

같은 모델 패밀리, 양자화 보정 후 동급 품질인데
작업 종류에 따라 승패가 완전히 뒤집힙니다.

---

## tok/s가 거짓말하는 구조적 이유

응답을 받기까지의 총 시간을 분해하면 이렇습니다.

```
T_response = T_prompt_processing + T_thinking + T_generation
```

tok/s는 마지막 항(T_generation)만 측정합니다.
나머지 두 항이 체감 속도를 결정적으로 바꿉니다.

### 1. Prefill — 프롬프트 처리 시간

토큰을 하나라도 생성하기 전에, 모델은 입력 전체를 먼저 읽어야 합니다.
이 단계는 **메모리 대역폭**(memory bandwidth)에 병목이 걸립니다.

| Node | Prefill Speed | 8K context prefill |
|------|--------------|-------------------|
| 5090 | ~12,000 tok/s | 0.7s |
| DGX | ~2,000 tok/s | 4.0s |

5090의 1,792 GB/s 메모리 대역폭은 DGX의 273 GB/s 대비 6배 빠릅니다.
32K 토큰 컨텍스트에서 DGX는 프롬프트만 읽는 데 16초가 걸립니다.
5090은 같은 작업을 2.7초에 끝냅니다.

짧은 대화형 인터랙션에서 5090이 압도적으로 빠른 이유가 여기에 있습니다.

### 2. Thinking 토큰 — 보이지 않는 비용

thinking이 켜진 상태에서 Qwen 3.6은 출력 토큰의 **60~90%를 내부 추론**(`<think>` 블록)에 소모합니다.
사용자에게 보이는 응답이 200토큰이어도 실제 생성은 2,000토큰일 수 있습니다.
체감 속도를 정확히 반영하려면 이렇게 계산해야 합니다.

```
Effective tok/s = visible_tokens / wall_clock_time
```

5090 (204 tok/s, thinking 오버헤드 90%):
- Raw: 204 tok/s
- Effective: **20.4 tok/s** (10%만 사용자에게 도달)

DGX (65 tok/s, 동일 비율):
- Raw: 65 tok/s
- Effective: **6.5 tok/s**

절대 비율은 같지만, 진짜 문제는 다른 곳에 있습니다.
DGX가 듀얼 파이프라인(빌더가 생성, 리뷰어가 병렬 검증)으로 돌면,
단일 GPU의 직렬 워크플로우보다 전체 파이프라인 시간이 짧아집니다.

### 3. 파이프라인 오버헤드 — 멀티스텝 작업

코딩 작업에서 제 DGX duo는 이렇게 동작합니다.

1. DGX1 (Qwen 3.5) — 코드 생성. 속도 최적화, thinking OFF
2. DGX2 (Qwen 3.6) — 리뷰 및 수정. thinking ON, 버그 포착

각 65 tok/s이지만 역할이 분리되어 있습니다.
리뷰어가 버그를 잡아주기 때문에, 단일 GPU에서 재시도 루프를 도는 시간을 절약합니다.

---

## 프레임워크: TTR, Effective tok/s, TCT

3노드에서 수백 개의 태스크를 측정한 결과,
사용자 만족도를 실제로 예측하는 3가지 지표를 정의했습니다.

### TTR (Time to Response) — 응답까지 걸린 시간

```
TTR = T_prefill + T_thinking + T_generation
```

Enter를 누른 순간부터 완전한 응답이 보일 때까지의 월클 시간입니다.
사용자가 실제로 "느끼는" 속도입니다.

**예시 — 간단한 팩트 질문:**

| Node | Prefill | Thinking | Generation | **TTR** |
|------|---------|----------|------------|---------|
| 5090 | 0.3s | 0s (OFF) | 0.9s | **1.2s** |
| DGX | 2.1s | 0s (OFF) | 6.4s | **8.5s** |

tok/s 차이는 3.1배인데, TTR 차이는 7배입니다.
짧은 대화에서는 prefill 속도(메모리 대역폭)가 체감을 지배합니다.

### Effective tok/s — 유효 생성 속도

```
Effective tok/s = content_tokens / TTR
```

사용자에게 도달하는 토큰만 셉니다.
thinking 오버헤드를 제거한 실질 속도입니다.

| Node | Raw tok/s | Thinking overhead | **Effective tok/s** |
|------|-----------|-------------------|---------------------|
| 5090 (thinking OFF) | 204 | 0% | **204** |
| 5090 (thinking ON) | 204 | 85% | **~31** |
| DGX (thinking ON) | 65 | 85% | **~10** |

### TCT (Task Completion Time) — 작업 완료 시간

```
TCT = Σ(all TTR steps) + pipeline_overhead
```

멀티스텝 작업(코드 생성 -> 테스트 -> 수정 -> 검증)에서 전체 워크플로우를 포착합니다.
tok/s가 빨라도 3번 재시도하면, 느리지만 한 번에 맞추는 쪽에 집니다.

**예시 — 코딩 작업 (state machine, 3턴):**

| Setup | Pass 1 | Pass 2 | Pass 3 | **TCT** | Quality |
|-------|--------|--------|--------|---------|---------|
| 5090 solo | 15s | 12s | 18s | **45s** | 97.1 |
| DGX duo (build+review) | 22s | 16s | — | **38s** | 100.0 |

DGX duo가 더 빨리 끝났고, 품질 점수도 더 높습니다.
리뷰어가 버그를 잡아서 5090에서 필요했을 세 번째 패스를 생략했기 때문입니다.

---

## 작업별 최적 노드 — 실측 결과

3노드 55개 태스크(Local-Trinity 벤치마크)를 돌린 결과입니다.

| Task Type | Winner | Why |
|-----------|--------|-----|
| Single-turn factual | **5090** | Prefill 속도가 지배. TTR 7배 우위. |
| Simple code generation | **5090** | Thinking OFF, 순수 생성 속도 승. |
| Complex multi-turn coding | **DGX duo** | 리뷰어가 버그 포착, 재시도 루프 회피. |
| Long-context analysis | **DGX** | 128GB VRAM, 262K context. 5090은 32K 제한. |
| Math with reasoning | **5090** | 동일 품질에서 thinking 3배 빠름. |

작업 종류에 따라 최적 지표가 달라집니다.
하나의 숫자로는 이 차이를 담을 수 없습니다.

---

## GPU 구매 시 실전 조언

**로컬 LLM 하드웨어를 고를 때:**

- **대화형 사용** (채팅, 빠른 질의응답): 메모리 대역폭을 최대화하십시오. 컨슈머 GPU(RTX 5090 등)가 유리합니다.
- **에이전틱 워크플로우** (멀티스텝, 도구 호출): VRAM과 컨텍스트 윈도우를 최대화하십시오. 워크스테이션/데이터센터 GPU가 유리합니다.
- **아키텍처 간 tok/s 비교는 무의미합니다** — 컨슈머 GPU의 200 tok/s Q4와 데이터센터 GPU의 65 tok/s FP8은 용도가 다릅니다.

**벤치마크를 만들거나 읽을 때:**

- 대화형 태스크에는 TTR을, 에이전틱 태스크에는 TCT를 보고하십시오
- Prefill 속도와 Generation 속도를 반드시 분리하여 기재하십시오
- thinking이 켜져 있으면 raw tok/s와 effective tok/s를 모두 표기하십시오
- 혼합 워크로드에서 단일 tok/s로 요약하지 마십시오

---

## 데이터 출처

모든 측정값은 Qwen 3.5/3.6 35B MoE를 운영하는 개인 3노드 홈랩에서 나왔습니다.

- **5090 harness**: 100+ scenarios, v1–v7, thinking ON/OFF A/B tests
- **DGX duo harness**: 254 tests, 14 E2E scenarios, n=3 stability verification
- **Local-Trinity**: 19 suites, 55 tasks, 290 unit tests — cross-node comparison framework
- **Latency metrics research**: [full methodology](https://github.com/ArkNill/claude-code-hidden-problem-analysis)

---

*개인 하드웨어에서 측정한 결과입니다. 모델 크기, 양자화 방식, 컨텍스트 길이, 워크로드 구성에 따라 결과는 달라질 수 있습니다.*
