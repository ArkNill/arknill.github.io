---
title: "양자화, 결정론, Thinking 토큰: 오픈소스 LLM 프로덕션 운용기"
date: 2026-05-01
description: "FP8이 프로덕션 마지노선입니다. Q4 MoE는 CJK에서 16%를 잃습니다. vLLM은 MTP 하에서 비결정적입니다. Thinking 토큰은 잘못된 태스크에서 예산의 90%를 소모합니다. Qwen 3.5/3.6 35B를 3노드에서 운용하며 얻은 실전 교훈입니다."
tags: ["llm", "quantization", "vllm", "inference", "local-llm"]
showToc: true
TocOpen: false
draft: true
---

저는 Qwen 3.5와 3.6 (35B MoE, 활성 파라미터 3B)을 3개 노드에서 프로덕션으로 운용하고 있습니다.
DGX Spark 2대(FP8, vLLM)와 RTX 5090 1대(Q4, llama.cpp) 구성입니다.
100개 이상의 벤치마크 시나리오와 수천 회의 추론 호출을 거치면서,
디버깅 시간의 대부분을 차지한 문제는 딱 세 가지였습니다.

1. **양자화 손실은 균일하지 않습니다** — MoE 모델의 Q4는 CJK 태스크에서 16%를 잃습니다
2. **vLLM은 비결정적입니다** — 동일한 프롬프트에서 다른 출력이 나옵니다
3. **Thinking 토큰은 이득 없는 태스크에서 예산의 60~90%를 먹습니다**

이 세 가지 모두 표준 벤치마크에서는 드러나지 않습니다.
그런데 프로덕션에서는 전부 문제를 일으킵니다.

---

## 1. 양자화: FP8이 프로덕션 마지노선입니다

"30B급 모델은 Q4에서 1% 미만 손실"이라는 통설이 있습니다.
**Dense 모델의 영어 벤치마크에서만 맞는 말입니다.**

### MoE와 Dense는 다릅니다

먼저 용어를 정리하겠습니다.

- **Dense 모델**: 모든 파라미터가 매 추론마다 활성화됩니다. 70B면 70B 전체가 동작합니다.
- **MoE (Mixture of Experts) 모델**: 전체 파라미터 중 일부 "전문가(Expert)"만 선택적으로 활성화됩니다. 35B 전체 중 3B만 동작하는 식입니다.

35B MoE에서 활성 파라미터가 3B라는 것은, 양자화 관점에서 사실상 소형 모델이라는 뜻입니다.
"총 35B"라는 숫자는 오해를 유발합니다.
실제로 양자화되는 것은 동적으로 라우팅되는 Expert 가중치이고,
라우터 로짓(router logit)에서 비트 하나가 뒤집히면 **이산적 오라우팅(discrete misrouting)**이 발생합니다.
Dense 모델처럼 부드럽게 성능이 떨어지는 것이 아니라, 아예 다른 Expert가 선택되는 것입니다.

25편 이상의 논문(Dettmers 2022, Ouyang 2024, APEX 2025)과 직접 실측한 결과입니다.

| Architecture | Q4 English Loss | Q4 CJK Loss | Mechanism |
|---|---|---|---|
| Dense 30B+ | <1% | 4–5% | Smooth degradation |
| **MoE 35B-A3B** | **1–14%** | **10–17%** | Router misrouting (discrete) |

### CJK 페널티: "잘 되는데?"의 함정

한국어/일본어/중국어 사용자라면 여기가 핵심입니다.
Q4로 돌려보고 "잘 되는데?"라고 느끼셨다면, 자동 벤치마크 점수도 같은 말을 할 것입니다.
문제는 사람이 직접 평가하면 결과가 완전히 다르다는 점입니다.

Marchisio et al. (EMNLP 2024) 연구에 따르면, **자동 벤치마크는 CJK 손실을 10~15 포인트 과소평가합니다.**

| Language | Auto Benchmark | Human Eval | Gap |
|----------|---------------|------------|-----|
| Japanese | -1.7% | **-16.0%** | 14.3pp |
| Korean | minor | **-4.6%** | — |
| Chinese (MGSM) | **-17.3%** | — | — |

제가 Qwen 3.5 35B-A3B에서 FP8 대비 Q4를 한국어 사실 기반 태스크로 측정한 결과,
**사람이 평가한 품질은 -16%** 하락했습니다.
같은 테스트에서 자동 메트릭은 -1.2%만 보여줬습니다.

많은 분들이 Q4로 운용하면서 "벤치마크 점수 괜찮은데?"라고 하시는데,
한국어/일본어 출력을 사람이 읽어보면 미묘한 문맥 오류, 조사 탈락, 의미 왜곡이 누적되어 있습니다.
자동 점수로는 잡히지 않는 부분입니다.

### Shared Expert 문제

MoE 아키텍처에는 "Shared Expert"가 있습니다.
라우팅과 무관하게 항상 활성화되는 공유 가중치입니다.
이 공유 가중치의 **첨도(kurtosis)가 13.10**으로, 라우팅 Expert(3.41)의 거의 4배입니다.
높은 첨도는 이상치(outlier) 값이 많다는 뜻이고,
저비트 양자화에서 이 값들이 잘려나갑니다(clipping).

APEX (2025)는 Expert별 차등 양자화(라우팅 Expert는 Q6_K, Shared Expert는 Q8_0)가 F16 품질과 동등함을 보였습니다.
모든 Expert에 균일 Q4를 적용하면 태스크에 따라 1~14% 손실이 발생합니다.

### 제 구성

```
DGX (production):  FP8 — quality baseline, 65 tok/s
5090 (interactive): Q4 — speed priority, 204 tok/s, accept CJK penalty
```

**FP8이 프로덕션 마지노선입니다.**
그 아래로 내려가면, 사용자가 한국어/일본어 출력 품질 저하를 체감할 때까지 드러나지 않는 방식으로 품질을 희생하는 것입니다.

---

## 2. vLLM 비결정성: MTP 문제

Local-Trinity 벤치마크(55개 태스크, 노드당 n=3)를 돌리면서 발견했습니다.
**vLLM은 동일 프롬프트에서 다른 출력을 생성합니다.**

| Node | Backend | Stability (n=3) |
|------|---------|-----------------|
| Desktop 5090 | llama-server | **STABLE 12/12** |
| DGX2 | vLLM + MTP | **UNSTABLE 4/12** |

같은 Qwen 3.6 모델, 같은 프롬프트입니다.
5090은 모든 태스크에서 3회 실행 결과가 동일했습니다.
DGX는 심하게 흔들렸습니다 — 한 태스크가 3회 실행에서 100, 100, 9를 기록했습니다.

### 원인: 삼중 비결정성

vLLM에는 복합적으로 작용하는 세 가지 비결정성 원인이 있습니다.

1. **CUDA 커널 비결정성** — FP8 행렬 곱셈에서 누적 순서가 스레드 스케줄링에 따라 달라집니다
2. **배치 스케줄링** — Continuous batching으로 인해 요청 인터리빙이 매 실행마다 다릅니다
3. **MTP (Multi-Token Prediction) 투기적 디코딩** — 투기 토큰의 수락/거부가 타이밍에 따라 달라집니다

MTP는 여러 토큰을 동시에 예측하여 속도를 높이는 기법입니다.
추측이 맞으면 채택하고, 틀리면 버립니다.
이 "맞다/틀리다" 판정이 실행 타이밍에 의존하기 때문에 결과가 매번 달라집니다.

Thinking Machines Lab이 대규모로 확인한 바에 따르면,
Qwen3-235B를 vLLM으로 1,000회 실행했을 때 동일 프롬프트에서 **80가지 서로 다른 출력**이 나왔습니다.

### MTP가 주범입니다

DGX2에서 MTP를 비활성화하고 테스트했습니다.

- **MTP ON**: UNSTABLE 4/12, HMT-03이 한 번은 100→39
- **MTP OFF**: STABLE, HMT-03 3/3 통과 (std 0.15s)

하지만 MTP OFF는 14% 속도 손실(61→52.7 tok/s)을 수반하고,
Thinking 중심 태스크에서 타임아웃이 발생했습니다.

### 운용상 함의

```
Production: MTP ON + n=3 mean scoring (accept variance)
Single-shot trust: Desktop 5090 (llama-server) only
```

결정적(deterministic) 출력이 필요한 상황(테스트 스위트, 재현 가능한 연구)이라면,
**vLLM + MTP 조합을 쓰지 마십시오.**
llama.cpp를 쓰거나 MTP를 비활성화하고 속도 손실을 감수해야 합니다.

---

## 3. Thinking 토큰: "더 똑똑하게"가 오히려 나빠지는 경우

Qwen 3.6은 요청별 Thinking 제어를 지원합니다.
모든 태스크 유형에서 Thinking ON과 OFF를 비교했습니다.

| Task Type | Thinking ON | Thinking OFF | Speed Difference |
|-----------|-------------|--------------|-----------------|
| Factual queries | 9/9 | 9/9 | OFF 2.5x faster |
| Code generation | 5/5 | 5/5 | OFF 4.2x faster |
| Code debugging | 3/3 | 3/3 | OFF 6.0x faster |
| Multi-turn coding | **1/2** | **2/2** | OFF 7.3x faster |
| Complex math | **3/3** | **0/3** | ON required |

**코딩 태스크에서는 Thinking OFF가 정답입니다.**
동등하거나 더 나은 결과를 4~7배 빠른 속도로 얻을 수 있습니다.
Thinking ON이 도움이 되는 것은 복잡한 수학적 추론뿐입니다.
그마저도 치명적인 실패 모드가 있습니다.

### Thinking 폭주(Runaway) 문제

복잡한 코딩 태스크에서 Thinking을 활성화하면,
Qwen 3.6이 **Thinking 루프**에 빠지는 경우가 있습니다.
전체 토큰 예산을 내부 추론에 소진하고, 실제 코드 출력에 쓸 용량이 남지 않습니다.

```
bugfix task:  TTFT 141s, completion_tokens=1, empty output (3/3 repro)
refactor task: TTFT 130s, completion_tokens=291, incomplete code
```

이것은 알려진 이슈(QwenLM/Qwen3.6#88)입니다.
어려운 코딩 태스크의 17.4%에서 Thinking 토큰 고갈이 발생합니다.
그 중 84%는 `<think>` 블록 내에서 반복 루프를 보입니다.

### 상황을 악화시키는 vLLM 버그

vLLM 이슈 [#39573](https://github.com/vllm-project/vllm/issues/39573)이 있습니다.
MTP가 활성화된 상태에서는 `thinking_token_budget`이 **조용히 무시됩니다.**
투기적 디코딩을 사용하면서 Thinking 토큰에 상한을 걸 수 없습니다.
파라미터는 에러 없이 받아들여지지만 아무 효과가 없습니다.

결과적으로:
- MTP ON + thinking ON = **예산 제어 불가**, 폭주 가능
- MTP OFF + thinking ON + budget = 동작하지만 14% 느림
- MTP ON + thinking OFF = **안전하고 가장 빠름** (코딩용)

### 요청별 제어

해결책은 요청별 동적 Thinking 제어입니다.

```json
// Default (all coding tasks):
{"chat_template_kwargs": {"enable_thinking": false}}

// Complex math only:
// Omit extra_body → server default (thinking ON)
```

서버 재시작이 필요 없습니다.
API 호출 단위로 Thinking을 토글합니다.
제 하네스는 태스크 복잡도를 사전에 분류하고 그에 따라 라우팅합니다.

---

## 정리: 프로덕션 구성

각 하네스의 v1~v7(노드당 100+ 시나리오, 290+ 유닛 테스트)을 거쳐
테스트를 생존한 구성입니다.

| Decision | Choice | Evidence |
|----------|--------|----------|
| Quantization floor | **FP8** | Q4 MoE: CJK -16%, router misrouting |
| Speed tier | **Q4 for interactive, FP8 for production** | Accept CJK penalty for 3x speed |
| vLLM determinism | **MTP ON + n=3 mean** | Single-shot determinism only with llama-server |
| Thinking default | **OFF** | ON only for complex math (3/3 vs 0/3) |
| Thinking budget | **Not available with MTP** | vLLM #39573 open |
| Max tokens (thinking ON) | **32768** | Prevents runaway (80% → 0% failure) |

이 결정들 중 논문을 읽거나 리더보드를 따라가서 나온 것은 하나도 없습니다.
전부 같은 태스크를 반복 실행하면서 실패 모드가 스스로 모습을 드러낼 때까지 돌린 결과입니다.

---

## References

- Dettmers & Zettlemoyer (2022): [LLM.int8()](https://arxiv.org/abs/2212.09720) — 35,000 experiments on quantization scaling
- Marchisio et al. (EMNLP 2024): CJK quality loss under quantization
- APEX (2025): Per-expert quantization for MoE models
- Thinking Machines Lab: vLLM non-determinism at scale (Qwen3-235B, 1000 runs)
- vLLM [#39573](https://github.com/vllm-project/vllm/issues/39573): MTP + thinking_budget incompatibility
- QwenLM [Qwen3.6#88](https://github.com/QwenLM/Qwen3.6/issues/88): Thinking token runaway on LiveCodeBench

---

*DGX Spark (128GB, vLLM 0.19.1)과 RTX 5090 (32GB, llama.cpp)에서 측정한 결과입니다. 구성에 따라 수치가 다를 수 있습니다.*
