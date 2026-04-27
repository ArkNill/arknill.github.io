---
title: "양자화, 결정론, Thinking 토큰: 오픈소스 LLM을 프로덕션에서 운용하기"
date: 2026-04-27
description: "FP8이 프로덕션 마지노선입니다. Q4 MoE는 CJK에서 16%를 잃습니다. vLLM은 MTP 하에서 비결정적입니다. Thinking 토큰은 잘못된 태스크에 예산의 90%를 소모합니다. Qwen 3.5/3.6 35B를 3노드에서 운용하며 얻은 실전 교훈입니다."
tags: ["llm", "quantization", "vllm", "inference", "local-llm"]
showToc: true
TocOpen: false
draft: false
---

저는 Qwen 3.5와 3.6 (35B MoE, 3B active parameters)을 3개 노드에 걸쳐 프로덕션으로 운용합니다 — DGX Spark 2대 (FP8, vLLM)와 RTX 5090 1대 (Q4, llama.cpp).
100회 이상의 벤치마크 시나리오와 수천 건의 추론 호출을 거친 후,
디버깅 시간의 대부분을 차지한 문제는 세 가지였습니다:

1. **양자화 손실**은 균일하지 않습니다 — MoE 모델을 Q4로 양자화하면 CJK 태스크에서 16%를 잃습니다
2. **vLLM은 비결정적입니다** — speculative decoding 하에서 동일한 프롬프트가 다른 출력을 생성합니다
3. **Thinking 토큰**은 아무런 이점이 없는 태스크에서 예산의 60–90%를 소모합니다

이 문제들은 표준 벤치마크에서 드러나지 않습니다.
그러나 모두 프로덕션 워크플로우를 깨뜨립니다.

---

## 1. 양자화: FP8이 프로덕션 마지노선입니다

일반적인 통념 — "30B+ 모델은 Q4에서 1% 미만의 손실" — 은 **Dense 모델의 영어 벤치마크에서만 사실**입니다.

### MoE 모델은 다릅니다

3B active parameters를 가진 35B MoE는 양자화 관점에서 사실상 작은 모델입니다.
"35B total"이라는 숫자는 오해를 불러옵니다 — 동적으로 라우팅되는 expert 가중치를 양자화하는 것이며,
router logits에서 발생하는 비트 오차는 Dense 모델처럼 완만한 열화가 아닌 **이산적 미스라우팅**을 유발합니다.

25편 이상의 논문 (Dettmers 2022, Ouyang 2024, APEX 2025)과 직접 측정 결과:

| Architecture | Q4 English Loss | Q4 CJK Loss | Mechanism |
|---|---|---|---|
| Dense 30B+ | <1% | 4–5% | Smooth degradation |
| **MoE 35B-A3B** | **1–14%** | **10–17%** | Router misrouting (discrete) |

### CJK 페널티

Marchisio et al. (EMNLP 2024)에 따르면 **자동 벤치마크는 사람 평가 대비 CJK 손실을 10–15 퍼센트포인트 과소평가**합니다:

| Language | Auto Benchmark | Human Eval | Gap |
|----------|---------------|------------|-----|
| Japanese | -1.7% | **-16.0%** | 14.3pp |
| Korean | minor | **-4.6%** | — |
| Chinese (MGSM) | **-17.3%** | — | — |

제가 Qwen 3.5 35B-A3B에서 측정한 결과: 한국어 팩트 태스크에서 FP8 대비 Q4는 **사람 평가 품질 기준 -16%** 였으나
자동 메트릭은 -1.2%만 표시했습니다.

### Shared Expert 문제

MoE 아키텍처에는 "shared experts" (라우팅과 무관하게 항상 활성화)가 존재합니다.
이 shared 가중치의 **kurtosis는 13.10** — routed experts (3.41) 대비 약 4배 높습니다.
높은 kurtosis는 저비트 양자화 시 클리핑되는 아웃라이어 값을 의미합니다.

APEX (2025)는 expert별 양자화 — routed experts에 Q6_K, shared experts에 Q8_0 — 가 F16 품질에 근접한다고 보여줬습니다.
모든 experts에 균일하게 Q4를 적용하면 태스크에 따라 1–14%를 잃습니다.

### 실제 운용 구성

```
DGX (production):  FP8 — quality baseline, 65 tok/s
5090 (interactive): Q4 — speed priority, 204 tok/s, accept CJK penalty
```

**FP8이 프로덕션 마지노선입니다.**
그 아래로 내려가면, 사용자가 한국어/일본어 출력 품질 저하를 체감할 때까지 드러나지 않는 방식으로 품질을 속도와 교환하게 됩니다.

---

## 2. vLLM 비결정성: MTP 문제

Local-Trinity 벤치마크 (55 tasks, 노드당 n=3) 과정에서,
**vLLM이 동일한 프롬프트로부터 서로 다른 출력을 생성한다**는 사실을 발견했습니다:

| Node | Backend | Stability (n=3) |
|------|---------|-----------------|
| Desktop 5090 | llama-server | **STABLE 12/12** |
| DGX2 | vLLM + MTP | **UNSTABLE 4/12** |

동일한 Qwen 3.6 모델, 동일한 프롬프트입니다.
5090은 모든 태스크에서 3회 실행 모두 동일한 출력을 생성했습니다.
DGX는 큰 편차를 보였습니다 — 한 태스크는 3회 실행에서 100, 100, 9점을 기록했습니다.

### 근본 원인: 삼중 비결정성

vLLM에는 복합적으로 작용하는 세 가지 비결정성 원인이 있습니다:

1. **CUDA kernel 비결정성** — FP8 행렬곱 누산 순서가 스레드 스케줄링에 따라 변동합니다
2. **Batch scheduling** — continuous batching으로 인해 매 실행마다 요청 인터리빙이 달라집니다
3. **MTP (Multi-Token Prediction) speculation** — 추측된 토큰의 수락/거절이 타이밍에 따라 달라집니다

Thinking Machines Lab이 대규모 환경에서 확인한 결과:
Qwen3-235B를 vLLM으로 1,000회 실행했을 때 동일 프롬프트에서 **80개의 서로 다른 출력 변형**이 나왔습니다.

### MTP가 주요 원인입니다

DGX2에서 MTP를 비활성화하고 테스트했습니다:
- **MTP ON**: UNSTABLE 4/12, HMT-03이 한 실행에서 100→39점
- **MTP OFF**: STABLE, HMT-03 3/3 통과 (std 0.15s)

다만 MTP OFF는 14% 속도 저하 (61→52.7 tok/s)를 수반하며
thinking이 무거운 태스크에서 타임아웃을 유발했습니다.

### 운영상 함의

```
Production: MTP ON + n=3 mean scoring (accept variance)
Single-shot trust: Desktop 5090 (llama-server) only
```

결정론적 출력이 필요한 경우 (테스트 스위트, 재현 가능한 연구), **vLLM + MTP를 사용하지 마십시오**.
llama.cpp를 사용하거나, MTP를 비활성화하고 속도 페널티를 수용하십시오.

---

## 3. Thinking 토큰: "더 똑똑한"이 오히려 해가 되는 경우

Qwen 3.6은 요청별 thinking 제어를 지원합니다.
모든 태스크 유형에서 thinking ON과 OFF를 비교 실행했습니다:

| Task Type | Thinking ON | Thinking OFF | Speed Difference |
|-----------|-------------|--------------|-----------------|
| Factual queries | 9/9 | 9/9 | OFF 2.5x faster |
| Code generation | 5/5 | 5/5 | OFF 4.2x faster |
| Code debugging | 3/3 | 3/3 | OFF 6.0x faster |
| Multi-turn coding | **1/2** | **2/2** | OFF 7.3x faster |
| Complex math | **3/3** | **0/3** | ON required |

**코딩에서 Thinking OFF가 올바른 기본값입니다.**
동등하거나 더 나은 결과를 4–7배 빠른 속도로 생성합니다.
Thinking ON은 복잡한 수학적 추론에서만 도움이 되며 — 그 경우에도 치명적 실패 모드가 존재합니다.

### Thinking 폭주 문제

복잡한 코딩 태스크에서 thinking을 활성화하면,
Qwen 3.6은 때때로 **thinking 루프**에 빠집니다 — 전체 토큰 예산을 내부 추론에 소모하여
실제 코드 출력을 위한 용량이 남지 않습니다:

```
bugfix task:  TTFT 141s, completion_tokens=1, empty output (3/3 repro)
refactor task: TTFT 130s, completion_tokens=291, incomplete code
```

이는 알려진 이슈입니다 (QwenLM/Qwen3.6#88): 어려운 코딩 태스크의 17.4%에서 thinking 토큰 고갈이 발생합니다.
그 중 84%는 `<think>` 블록 내에서 반복 루프를 보입니다.

### 상황을 악화시키는 vLLM 버그

vLLM issue [#39573](https://github.com/vllm-project/vllm/issues/39573): MTP가 활성화된 상태에서
`thinking_token_budget`이 **조용히 무시됩니다**.
Speculative decoding을 사용하면서 thinking 토큰에 상한을 둘 수 없습니다.
파라미터는 에러 없이 수용되지만 아무 동작도 하지 않습니다.

이는 다음을 의미합니다:
- MTP ON + thinking ON = **예산 제어 불가**, 폭주 가능
- MTP OFF + thinking ON + budget = 동작하나 14% 느림
- MTP ON + thinking OFF = **안전하고 가장 빠름** (코딩용)

### 요청별 제어

해결책은 동적 요청별 thinking 제어입니다:

```json
// Default (all coding tasks):
{"chat_template_kwargs": {"enable_thinking": false}}

// Complex math only:
// Omit extra_body → server default (thinking ON)
```

서버 재시작이 필요 없습니다.
Thinking은 API 호출 단위로 토글됩니다.
제 하네스는 태스크 복잡도를 사전 분류하여 그에 맞게 라우팅합니다.

---

## 요약: 프로덕션 구성

각 하네스의 v1–v7 (노드당 100+ 시나리오, 290+ 단위 테스트)을 거친 후,
테스트를 통과하여 살아남은 구성은 다음과 같습니다:

| Decision | Choice | Evidence |
|----------|--------|----------|
| Quantization floor | **FP8** | Q4 MoE: CJK -16%, router misrouting |
| Speed tier | **Q4 for interactive, FP8 for production** | Accept CJK penalty for 3x speed |
| vLLM determinism | **MTP ON + n=3 mean** | Single-shot determinism only with llama-server |
| Thinking default | **OFF** | ON only for complex math (3/3 vs 0/3) |
| Thinking budget | **Not available with MTP** | vLLM #39573 open |
| Max tokens (thinking ON) | **32768** | Prevents runaway (80% → 0% failure) |

이 결정들 중 어느 것도 논문을 읽거나 리더보드를 따라해서 나온 것이 아닙니다.
모두 동일한 태스크를 반복 실행하여 실패 모드가 저절로 드러날 때까지 기다린 결과입니다.

---

## 참고 문헌

- Dettmers & Zettlemoyer (2022): [LLM.int8()](https://arxiv.org/abs/2212.09720) — 35,000 experiments on quantization scaling
- Marchisio et al. (EMNLP 2024): CJK quality loss under quantization
- APEX (2025): Per-expert quantization for MoE models
- Thinking Machines Lab: vLLM non-determinism at scale (Qwen3-235B, 1000 runs)
- vLLM [#39573](https://github.com/vllm-project/vllm/issues/39573): MTP + thinking_budget incompatibility
- QwenLM [Qwen3.6#88](https://github.com/QwenLM/Qwen3.6/issues/88): Thinking token runaway on LiveCodeBench

---

*DGX Spark (128GB, vLLM 0.19.1)와 RTX 5090 (32GB, llama.cpp)에서 측정했습니다. 사용자의 구성에 따라 결과가 달라질 수 있습니다.*
