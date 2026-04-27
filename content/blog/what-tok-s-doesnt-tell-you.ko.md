---
title: "tok/s가 알려주지 않는 것: 실제로 체감되는 LLM 속도 측정법"
date: 2026-04-27
description: "204 tok/s GPU가 65 tok/s GPU보다 느리게 느껴지는 작업이 있습니다. tok/s만으로는 오해를 부릅니다 — 사용자가 실제로 체감하는 속도를 측정하는 프레임워크(TTR, Effective tok/s, TCT)를 소개합니다."
tags: ["llm", "benchmarks", "inference", "local-llm"]
showToc: true
TocOpen: false
draft: false
---

저는 Qwen 3.6 35B를 세 대의 머신에서 운영하고 있습니다.
RTX 5090은 **204 tok/s**로 생성합니다.
DGX Spark 페어는 **65 tok/s**로 생성합니다.
벤치마크 리더보드의 모든 지표 기준으로, 5090이 3배 빠릅니다.

그런데 thinking이 켜진 다단계 코딩 작업에서는,
DGX 페어가 **더 빨리 작업을 완료**합니다.
반대로 단일 턴 질문에서는,
5090이 2초 이내에 답변을 전달하는 반면 DGX는 8–12초가 걸립니다.

tok/s만으로는 실제 사용자 경험에 대해 아무것도 알 수 없었습니다.
세 노드 모두를 위한 벤치마크를 만들면서 배운 것을 공유합니다.

---

## 역설

제 3-노드 구성입니다:

| Node | GPU | Model | tok/s | Memory BW |
|------|-----|-------|-------|-----------|
| Desktop 5090 | RTX 5090 (32GB) | Qwen3.6-35B MoE Q4 | **204** | 1,792 GB/s |
| DGX1 | Grace Hopper (128GB) | Qwen3.5-35B FP8 | 65 | 273 GB/s |
| DGX2 | Grace Hopper (128GB) | Qwen3.6-35B FP8 | 65 | 273 GB/s |

5090이 생성 속도에서 3.1배 빠릅니다. 하지만:

- **단순한 사실 확인 질문**: 5090은 1.2초에 응답합니다. DGX는 8.5초 걸립니다. 실제 경과 시간 기준으로 5090이 7배 앞섭니다.
- **thinking이 켜진 다중 턴 코딩 작업**: 5090은 45초에 완료합니다. DGX 페어(builder + reviewer)는 38초에 더 높은 품질로 완료합니다.

같은 모델 패밀리, 동일한 양자화 보정 품질입니다.
작업 유형에 따라 결과가 역전됩니다.

---

## tok/s가 거짓말하는 이유

응답을 받기까지의 총 시간은 다음과 같습니다:

```
T_response = T_prompt_processing + T_thinking + T_generation
```

tok/s는 마지막 항만 측정합니다.
다음은 tok/s가 놓치는 것들입니다:

### 1. 프롬프트 처리 (Prefill)

단일 토큰을 생성하기 전에, 모델은 전체 입력을 처리해야 합니다.
이 과정은 연산량이 아니라 **메모리 대역폭에 의해 좌우**됩니다.

| Node | Prefill Speed | 8K context prefill |
|------|--------------|-------------------|
| 5090 | ~12,000 tok/s | 0.7s |
| DGX | ~2,000 tok/s | 4.0s |

5090의 1,792 GB/s 메모리 대역폭은 DGX의 273 GB/s 대비 프롬프트를 6배 빠르게 처리합니다.
긴 컨텍스트 윈도우(32K 토큰)의 경우,
DGX는 프롬프트를 읽는 데만 16초를 소비합니다.
5090은 2.7초면 됩니다.

### 2. Thinking 토큰 (보이지 않는 비용)

thinking이 켜져 있으면,
Qwen 3.6은 **출력 토큰의 60–90%를 사용자에게 전달되지 않는 내부 추론에 사용**합니다.
응답에 200개의 보이는 토큰이 표시되지만 실제로는 2,000개를 생성했다면(`<think>` 블록에 1,800개),
유효 속도는 다음과 같습니다:

```
Effective tok/s = visible_tokens / wall_clock_time
```

204 tok/s의 5090에서 90% thinking 오버헤드가 있을 때:
- Raw: 204 tok/s
- Effective: **20.4 tok/s** (10%만 보이는 출력이 됩니다)

65 tok/s의 DGX에서 동일한 thinking 비율일 때:
- Raw: 65 tok/s
- Effective: **6.5 tok/s**

5090이 절대적 수치에서는 여전히 이기지만, 격차는 3.1x에서 3.1x로 유지됩니다.
진짜 문제는: DGX가 듀얼 모델 파이프라인(builder가 생성하고, reviewer가 병렬 검증)을 사용하면,
전체 파이프라인 시간이 단일 모델 직렬 워크플로우를 이길 수 있다는 것입니다.

### 3. 파이프라인 오버헤드 (다단계 작업)

코딩 작업에서 제 DGX duo는 다음과 같이 동작합니다:
1. DGX1 (Qwen 3.5)이 코드를 생성합니다 — 속도에 최적화, thinking OFF
2. DGX2 (Qwen 3.6)가 리뷰하고 수정합니다 — thinking ON, 버그를 잡습니다

각 65 tok/s인 두 패스이지만, 특화된 역할을 수행합니다.
리뷰어가 단일 패스 생성기가 놓치는 오류를 잡아내어 —
단일 GPU 구성에서 시간을 소모하는 재시도 루프를 절약합니다.

---

## 프레임워크: TTR, Effective tok/s, TCT

세 노드 모두에서 수백 개의 작업을 측정한 후,
사용자 만족도를 실제로 예측하는 세 가지 지표를 정의했습니다:

### TTR (Time to Response)

```
TTR = T_prefill + T_thinking + T_generation
```

Enter 키를 누른 시점부터 전체 응답을 보는 시점까지의 실제 경과 시간입니다.
이것이 사용자가 체감하는 것입니다.

**예시 — 단순한 사실 확인 질문:**

| Node | Prefill | Thinking | Generation | **TTR** |
|------|---------|----------|------------|---------|
| 5090 | 0.3s | 0s (OFF) | 0.9s | **1.2s** |
| DGX | 2.1s | 0s (OFF) | 6.4s | **8.5s** |

5090은 tok/s에서 3.1배 빠를 뿐인데, TTR에서는 7배 빠릅니다.
차이는 prefill에 있습니다 — 메모리 대역폭이 짧은 인터랙션을 지배합니다.

### Effective tok/s

```
Effective tok/s = content_tokens / TTR
```

사용자에게 도달하는 토큰만 계산합니다.
thinking 오버헤드를 제거합니다.

| Node | Raw tok/s | Thinking overhead | **Effective tok/s** |
|------|-----------|-------------------|---------------------|
| 5090 (thinking OFF) | 204 | 0% | **204** |
| 5090 (thinking ON) | 204 | 85% | **~31** |
| DGX (thinking ON) | 65 | 85% | **~10** |

### TCT (Task Completion Time)

```
TCT = Σ(all TTR steps) + pipeline_overhead
```

다단계 작업(코드 → 테스트 → 수정 → 검증)에서, TCT는 전체 워크플로우를 포착합니다.
더 빠른 tok/s라도 3번의 재시도가 필요하면, 한 번에 맞추는 느린 tok/s에 집니다.

**예시 — 코딩 작업 (상태 머신, 3턴):**

| Setup | Pass 1 | Pass 2 | Pass 3 | **TCT** | Quality |
|-------|--------|--------|--------|---------|---------|
| 5090 solo | 15s | 12s | 18s | **45s** | 97.1 |
| DGX duo (build+review) | 22s | 16s | — | **38s** | 100.0 |

DGX duo가 더 빨리 완료했고 *동시에* 더 높은 점수를 받았습니다 —
리뷰어가 5090에서는 세 번째 패스가 필요했을 버그를 잡아냈기 때문입니다.

---

## 각 노드가 이기는 시점

세 노드 모두에서 55개 작업을 실행한 결과입니다 (Local-Trinity 벤치마크 스위트):

| Task Type | Winner | Why |
|-----------|--------|-----|
| Single-turn factual | **5090** | Prefill 속도가 지배합니다. 7x TTR 우위. |
| Simple code generation | **5090** | Thinking OFF, 순수 생성 속도. |
| Complex multi-turn coding | **DGX duo** | 리뷰어가 버그를 잡아 재시도 루프를 회피합니다. |
| Long-context analysis | **DGX** | 128GB VRAM, 262K context. 5090은 32K 제한. |
| Math with reasoning | **5090** | 동일 품질, 3x 빠른 thinking. |

올바른 지표는 워크로드에 따라 달라집니다.
단일 숫자로는 이것을 포착할 수 없습니다.

---

## 실용적 시사점

**로컬 LLM용 하드웨어를 구매하신다면:**
- **인터랙티브 사용** (채팅, 빠른 질문): 메모리 대역폭을 극대화하십시오. 컨슈머 GPU가 유리합니다.
- **에이전틱 워크플로우** (다단계, 도구 사용): VRAM과 컨텍스트 윈도우를 극대화하십시오. 워크스테이션/데이터센터 GPU가 유리합니다.
- **아키텍처 간 tok/s를 비교하지 마십시오** — 컨슈머 GPU의 200 tok/s Q4 모델과 데이터센터 GPU의 65 tok/s FP8 모델은 용도가 다릅니다.

**벤치마크를 만드신다면:**
- 인터랙티브 작업에는 TTR을, 에이전틱 작업에는 TCT를 보고하십시오
- Prefill 속도와 생성 속도를 항상 분리하여 보고하십시오
- thinking이 켜져 있으면, raw와 effective tok/s를 모두 보고하십시오
- 혼합 워크로드에 대해 단일 tok/s 수치를 집계하지 마십시오

---

## 데이터 출처

모든 측정은 Qwen 3.5/3.6 35B MoE를 운영하는 제 3-노드 홈랩에서 수행되었습니다:
- **5090 harness**: 100+ 시나리오, v1–v7, thinking ON/OFF A/B 테스트
- **DGX duo harness**: 254 tests, 14 E2E 시나리오, n=3 안정성 검증
- **Local-Trinity**: 19 suites, 55 tasks, 290 unit tests — 크로스 노드 비교 프레임워크
- **Latency metrics research**: [전체 방법론](https://github.com/ArkNill/claude-code-hidden-problem-analysis)

---

*개인 하드웨어에서 측정된 결과입니다. 모델 크기, 양자화, 컨텍스트 길이, 워크로드 구성에 따라 결과는 달라질 수 있습니다.*
