---
title: "Claude Code와 로컬 35B 모델 비교 테스트: 크로스체크 하네스 구축기"
date: 2026-04-27
description: "Claude Code와 Codex를 로컬 Qwen 35B 모델과 비교하기 위해 세 가지 벤치마크 하네스를 구축했습니다. 하네스가 모델보다 버그가 더 많았습니다. v1→v7까지의 여정 — 55개 태스크, 290개 테스트, 그리고 'ALL_FAIL 7→0'이 평가에 대해 가르쳐 준 것들입니다."
tags: ["llm", "benchmarks", "claude-code", "local-llm", "testing"]
showToc: true
TocOpen: false
draft: false
---

저는 Claude Code (Opus 4.6)를 주력 코딩 도구로 사용하며 월 $200를 지불하고 있습니다.
동시에 두 대의 DGX Spark와 RTX 5090에서 Qwen 3.5/3.6 35B를 로컬로 운영하고 있습니다.
자연스러운 질문이 떠오릅니다: **로컬 35B 모델이 비용을 지불하고 있는 상용 도구와 비교하면 어떨까?**

이를 확인하기 위해 10일에 걸쳐 세 가지 별도의 벤치마크 하네스를 구축했습니다.
이 여정은 모델 자체보다 평가 방법론에 대해 더 많은 것을 가르쳐 주었습니다 — 왜냐하면 **하네스가 모델보다 버그가 더 많았기 때문입니다**.

---

## 세 가지 하네스

| 하네스 | 노드 | 초점 | 테스트 |
|---------|-------|-------|-------|
| **cc-crosscheck** | DGX1 (3.5) | CC vs Codex vs Local — 동일 태스크, 세 도구 | 254 |
| **dgx-duo** | DGX1 + DGX2 | Builder (3.5) + Reviewer (3.6) 페어 모드 | 254 |
| **Local-Trinity** | 전체 3노드 | 통합 크로스 노드 비교 | 290 |

세 하네스 모두 제로 의존성 Python (stdlib + urllib만 사용)으로 구성됩니다.
프레임워크 없음, 추론 노드에 pip 설치 없음.

---

## cc-crosscheck: 3-Peer 프로토콜

핵심 아이디어: 동일한 코딩 태스크를 세 "피어"에서 실행합니다 — Claude Code (Opus 4.6),
Codex (GPT-5.3), 그리고 로컬 35B 모델 — 그런 다음 출력을 비교합니다.

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

검증기는 단순히 "실행되는가"만 확인하지 않습니다 — AST 합의 분석을 수행하여
세 솔루션 간의 구조적 패턴을 비교합니다.
셋 중 둘이 특정 접근법에 동의하고 하나가 다른 경우, 그 차이가 플래그됩니다.

### 결과 (v0.9.6, 8 시나리오)

| 시나리오 | CC (Opus 4.6) | Local (Qwen 3.5) | 비고 |
|----------|---------------|-------------------|-------|
| LRU Cache | PASS | PASS | 구조적 합의 |
| Plugin Registry | PASS | PASS | — |
| Code Review | PASS | PASS | — |
| Race Condition Fix | PASS | PASS | — |
| Dijkstra | PASS | PASS | — |
| Data Pipeline | PASS | PASS | — |
| Config Merger | PASS | PASS (98.7) | 미미한 차이: nested validation 엣지 케이스 |
| Cache Refactor | PASS | PASS | — |

**결과: 24/24 ALL PASSED (n=3), 평균 점수 99.6.**

로컬 35B가 8개 코딩 시나리오 전체에서 Claude Code 품질에 일치했습니다.
격차는 단일 태스크 품질에 있지 않습니다 — 속도(CC는 최적화된 인프라 덕분에 더 빠르게 응답)와
상용 도구의 더 긴 컨텍스트 및 도구 사용 통합이 중요한 모호하고 다단계 태스크 처리에서 차이가 납니다.

---

## dgx-duo: 두 모델이 하나보다 나을 때

상용 도구와의 비교 대신,
dgx-duo는 이렇게 묻습니다: **두 개의 로컬 모델이 협력하면 하나보다 더 나은 결과를 낼 수 있을까?**

프로토콜:
1. **DGX1 (Qwen 3.5, thinking OFF)** — Builder. 빠르게 코드를 생성합니다.
2. **DGX2 (Qwen 3.6, thinking ON)** — Reviewer. 코드를 읽고, 버그를 찾고, 수정을 제안합니다.
3. 리뷰어가 문제를 발견하면 → 빌더가 피드백을 받아 두 번째 패스를 수행합니다.

### 결과 (v0.9.6, 14 E2E 시나리오)

| 모드 | 시나리오 | 통과율 | 평균 점수 |
|------|-----------|-----------|------------|
| Single (DGX1 단독) | 8 | 75% | 89.2 |
| **Pair (builder + reviewer)** | 8 | **100%** | **99.6** |
| Multi-step | 6 | 83% | 96.1 |

리뷰어가 단일 패스 생성기가 놓치는 버그를 잡아냅니다.
가장 흔히 잡아내는 항목들:
- 누락된 에러 처리 (pytest.raises 커버리지)
- 동시성 코드의 엣지 케이스 (레이스 컨디션)
- 불완전한 인터페이스 구현

### 고무도장 문제 (v0.9.3→v0.9.4)

초기 버전에는 치명적 결함이 있었습니다: 리뷰어가 **항상 승인했습니다**.
코드에 명백한 버그가 있어도 "좋아 보입니다"라고 말했습니다.
이것은 모델의 한계가 아닙니다 — 프롬프트 엔지니어링 실패입니다.

수정 (v0.9.4): 리뷰어의 역할을 "이 코드가 작동하는지 확인하라"에서 "이 코드를 리뷰하라 — 당신은 실패 판정을 내릴 권한이 있다"로 변경했습니다.
모델에게 거부할 수 있는 명시적 권한을 부여하자 행동이 완전히 변했습니다.
고무도장 비율이 >90%에서 <10%로 떨어졌습니다.

---

## Local-Trinity: 통합 벤치마크

Local-Trinity는 세 노드 전체를 하나의 벤치마크 스위트로 통합합니다.
동일 태스크, 세 노드 모두, 안정성을 위해 n=3:

### 아키텍처

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

### 태스크 카테고리

| 카테고리 | 스위트 | 태스크 | 예시 |
|----------|--------|-------|----------|
| Factual (KO/EN) | 5 | 16 | 한국어 상식, 영어 팩트, 모호한 질문 |
| Math | 2 | 7 | 산술, 모듈러, 조합론 |
| Coding | 6 | 14 | 단일 파일, 디버그, 멀티턴, 알고리즘 |
| Hard | 4 | 12 | 3턴 상태 머신, B-Tree, 정규식 엔진 |
| Agent | 2 | 6 | 속도 중심, 알고리즘 중심 |

### v1→v7 여정

실질적 학습이 일어난 지점입니다:

| 버전 | ALL_FAIL | 무엇이 깨졌나 | 근본 원인 |
|---------|----------|-----------|------------|
| **v1** | **7** | 모델이 분명히 풀 수 있는 태스크를 "실패" | 하네스 버그: 잘못된 assertion, 깨진 스코어링 |
| v2 | 1 | 여전히 하나의 지속적 실패 | Q4에게 난이도 과다 |
| v3 | 2 | "개선"으로 인한 회귀 | AST 스코어링이 새 버그 도입 |
| v4 | 2 | 동일한 두 개가 해결 불가 | try_pass 카운팅 오류 |
| **v5** | **0** | 드디어 전체 통과 | atexit 카운팅, 프롬프트 재작성 |
| v6 | 0 | 안정 — 기능 추가 | 성능 타이밍, thinking A/B |
| **v7** | **0** | 안정 — 최종 다듬기 | assert_total 분모 수정 |

**하네스가 모델보다 버그가 더 많았습니다.**
모델이 실패한다고 생각할 때마다, 실제 문제는 다음과 같았습니다:
- 유효한 대안 포맷을 거부하는 assert 매칭
- 정확하지만 다른 접근법에 패널티를 부여하는 스코어링
- thinking이 많은 태스크에 대해 너무 짧은 타임아웃
- 모델에게는 모호하지만 제게는 명확한 프롬프트

### v5 벤치마크 결과 (12 Hard 태스크, n=3)

| 태스크 | 5090 (Q4) | DGX1 (3.5 FP8) | DGX2 (3.6 FP8) | 판정 |
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

MAJORITY 케이스가 실제 모델 차이를 드러냅니다:
- **DGX1 (Qwen 3.5)** 은 상태 추적이 필요한 멀티턴 태스크에서 어려움을 겪습니다
- **B-Tree**: DGX1이 노드 분할에서 일관되게 실패합니다 (구조적 한계)
- **Code Review**: 5090과 DGX2 모두 pickle 보안 검사를 놓칩니다 (비결정적)

---

## 배운 것들

### 1. 벤치마크 품질 > 모델 품질

모델은 v1부터 괜찮았습니다.
제 평가 프레임워크가 깨져 있었습니다.
벤치마크가 모델이 분명히 처리할 수 있는 태스크에서 "실패"한다고 보여준다면, **버그는 벤치마크에 있습니다**.

### 2. 점수 ≠ 품질

100.0점은 "모든 구조적 검사를 통과했다"는 의미입니다 — "좋은 코드를 생성했다"가 아닙니다.
코드가 100.0점을 받았지만 검증기가 잡지 못하는 미묘한 버그(nested validation 엣지 케이스)가 있는 경우를 발견했습니다.
검증기는 측정하도록 설계된 것만 측정합니다. 여러분이 측정한다고 생각하는 것을 측정하지 않습니다.

### 3. n=3은 필수입니다

n=1에서는 모델 한계와 추론 비결정성을 구분할 수 없습니다.
vLLM의 MTP는 네 번 중 한 번꼴로 완전히 다른 출력을 생성합니다.
n=3 평균 스코어링이 신호와 노이즈를 분리합니다.

### 4. 35B 최적 지점

명확한 명세가 있는 코딩 태스크에서,
Qwen 35B MoE (3B active) FP8은 품질 면에서 상용 도구와 동등합니다.
격차가 나타나는 지점:
- 모호한 다단계 태스크 (상용 도구가 더 나은 지시 따르기를 보여줌)
- 롱 컨텍스트 검색 (128GB vs 클라우드 측 1M 토큰)
- 도구 사용 및 파일 시스템 상호작용 (Claude Code의 통합 레이어)

그 외 모든 것 — 함수 작성, 디버깅, 알고리즘, 리팩토링 —
에서는 로컬 모델이 작동합니다.

---

## 코드

세 하네스 모두 제로 의존성 Python입니다:
- **cc-crosscheck**: `~/GitHub/cc-crosscheck/` — 254 tests, dual-35B architecture
- **dgx-duo**: `~/GitHub/dgx-duo/` — 9 modules, 254 tests, pair mode
- **Local-Trinity**: `~/GitHub/local-trinity/` — 17 modules, 290 tests, 3-node unified

문서 정리 후 공개 릴리스 예정입니다.

---

*개인 하드웨어에서의 개인 벤치마크입니다.
모델: Qwen 3.5/3.6 35B-A3B (FP8 및 Q4).
상용 비교 대상: Claude Code with Opus 4.6 (v2.1.109).
Anthropic, OpenAI, Alibaba와 제휴 관계가 없습니다.*
