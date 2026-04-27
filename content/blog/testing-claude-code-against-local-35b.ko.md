---
title: "Claude Code vs 로컬 35B 모델: 크로스체크 하네스를 만들며 배운 것들"
date: 2026-05-03
description: "월 20만원짜리 Claude Code를 로컬 Qwen 35B가 대체할 수 있을까? 하네스 3개, 55개 태스크, 290개 테스트를 돌려봤습니다. 가장 놀라운 발견 — 하네스 버그가 모델 버그보다 많았습니다."
tags: ["llm", "benchmarks", "claude-code", "local-llm", "testing"]
showToc: true
TocOpen: false
draft: true
---

월 20만원 내고 쓰는 Claude Code, 로컬 35B 모델이 대체할 수 있을까?

저는 Claude Code (Opus 4.6)를 주력 코딩 도구로 사용합니다.
동시에 DGX Spark 2대와 RTX 5090에서 Qwen 3.5/3.6 35B를 로컬로 돌립니다.
매달 $200(약 27만원)을 내면서 자연스럽게 궁금해졌습니다.
**이 돈값을 하는 건가? 로컬 모델로 충분하지 않을까?**

답을 찾기 위해 10일간 벤치마크 하네스를 3개 만들었습니다.
결론부터 말하면 — 모델보다 **하네스가 더 많이 틀렸습니다**.
평가 도구를 만드는 과정에서, 모델 성능보다 평가 방법론에 대해 훨씬 더 많이 배웠습니다.

---

## 실험 설계: 세 가지 하네스

| 하네스 | 노드 | 초점 | 테스트 |
|---------|-------|-------|-------|
| **cc-crosscheck** | DGX1 (3.5) | CC vs Codex vs Local — 동일 태스크, 세 도구 | 254 |
| **dgx-duo** | DGX1 + DGX2 | Builder (3.5) + Reviewer (3.6) 페어 모드 | 254 |
| **Local-Trinity** | 전체 3노드 | 통합 크로스 노드 비교 | 290 |

세 하네스 모두 zero-dependency Python(표준 라이브러리 + urllib만 사용)입니다.
추론 노드에 pip install 한 번 없이 돌아갑니다.
프레임워크도 없습니다.

---

## cc-crosscheck: 동일 태스크를 세 도구로

핵심 아이디어는 단순합니다.
같은 코딩 문제를 Claude Code, Codex, 로컬 모델에 동시에 던지고 결과를 비교합니다.

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

검증기(Validator)는 단순히 "실행 가능한가"만 보지 않습니다.
AST(Abstract Syntax Tree) 합의 분석을 수행합니다.
세 솔루션의 구조적 패턴을 비교해서, 셋 중 둘이 같은 접근을 택했는데 하나만 다르면 그 차이를 플래그합니다.

### 결과 (v0.9.6, 8 시나리오)

| 시나리오 | CC (Opus 4.6) | Local (Qwen 3.5) | 비고 |
|----------|---------------|-------------------|-------|
| LRU Cache | PASS | PASS | Structural consensus |
| Plugin Registry | PASS | PASS | — |
| Code Review | PASS | PASS | — |
| Race Condition Fix | PASS | PASS | — |
| Dijkstra | PASS | PASS | — |
| Data Pipeline | PASS | PASS | — |
| Config Merger | PASS | PASS (98.7) | Minor: nested validation edge case |
| Cache Refactor | PASS | PASS | — |

**결과: 24/24 ALL PASSED (n=3), 평균 점수 99.6.**

8개 코딩 시나리오에서 로컬 35B가 Claude Code와 동등한 품질을 보였습니다.

"로컬 LLM은 아직 멀었다"는 인식과 달리, 명확한 스펙이 있는 코딩 태스크에서는 차이가 없었습니다.
차이가 나는 지점은 따로 있습니다 — 속도(CC의 최적화된 인프라), 그리고 모호한 다단계 태스크에서의 컨텍스트 활용입니다.

---

## dgx-duo: 두 모델이 협력하면?

상용 도구와의 비교 대신, 다른 질문을 던져봤습니다.
**로컬 모델 두 개가 협력하면 하나보다 나은 결과를 낼 수 있을까?**

프로토콜은 이렇습니다:
1. **DGX1 (Qwen 3.5, thinking OFF)** — Builder. 빠르게 코드를 생성합니다.
2. **DGX2 (Qwen 3.6, thinking ON)** — Reviewer. 코드를 읽고, 버그를 찾고, 수정을 제안합니다.
3. Reviewer가 문제를 발견하면 → Builder가 피드백을 받아 두 번째 패스를 수행합니다.

### 결과 (v0.9.6, 14 E2E 시나리오)

| 모드 | 시나리오 | 통과율 | 평균 점수 |
|------|-----------|-----------|------------|
| Single (DGX1 단독) | 8 | 75% | 89.2 |
| **Pair (builder + reviewer)** | 8 | **100%** | **99.6** |
| Multi-step | 6 | 83% | 96.1 |

혼자 돌리면 75%였던 통과율이, 리뷰어를 붙이니 100%가 됩니다.
리뷰어가 잡아내는 것들:
- 누락된 에러 처리 (pytest.raises 커버리지)
- 동시성 코드의 엣지 케이스 (race condition)
- 불완전한 인터페이스 구현

### 고무도장 문제 — "LGTM" 문화의 AI 버전 (v0.9.3→v0.9.4)

초기 버전에는 치명적인 결함이 있었습니다.
리뷰어가 **무조건 승인**했습니다.
코드에 명백한 버그가 있어도 "looks good"이라고 답했습니다.

개발자라면 익숙한 패턴일 겁니다.
PR 리뷰에서 "LGTM" 한 줄 달고 approve 누르는 문화.
AI도 똑같이 행동했습니다.

이건 모델의 한계가 아니라, 프롬프트 엔지니어링 실패였습니다.

수정 방법(v0.9.4): "이 코드가 작동하는지 확인하라" → "이 코드를 리뷰하라 — **실패 판정을 내릴 권한이 있다**"로 프롬프트를 변경했습니다.
모델에게 거부 권한을 명시적으로 부여하자 행동이 완전히 달라졌습니다.
고무도장 비율이 >90%에서 <10%로 떨어졌습니다.

교훈: LLM은 기본적으로 동의하려는 경향이 있습니다.
비판적 역할을 원한다면, 그 권한을 명시적으로 부여해야 합니다.

---

## Local-Trinity: 3노드 통합 벤치마크

Local-Trinity는 세 노드를 하나의 벤치마크 스위트로 통합한 최종 형태입니다.
동일 태스크를 세 노드에서 돌리고, 안정성을 위해 n=3(3회 반복)으로 측정합니다.

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
| Factual (KO/EN) | 5 | 16 | Korean trivia, English facts, ambiguous questions |
| Math | 2 | 7 | Arithmetic, modular, combinatorics |
| Coding | 6 | 14 | Single-file, debug, multi-turn, algorithms |
| Hard | 4 | 12 | 3-turn state machines, B-Trees, regex engines |
| Agent | 2 | 6 | Speed-focused, algorithm-heavy |

### v1→v7: 하네스 버그와의 전쟁

이 섹션이 이 글의 핵심입니다.

| 버전 | ALL_FAIL | 무엇이 깨졌나 | 근본 원인 |
|---------|----------|-----------|------------|
| **v1** | **7** | 모델이 분명히 풀 수 있는 태스크를 "실패" | 하네스 버그: 잘못된 assertion, 깨진 스코어링 |
| v2 | 1 | 여전히 하나의 지속적 실패 | Q4에게 난이도 과다 |
| v3 | 2 | "개선"으로 인한 회귀 | AST 스코어링이 새 버그 도입 |
| v4 | 2 | 동일한 두 개가 해결 불가 | try_pass 카운팅 오류 |
| **v5** | **0** | 드디어 전체 통과 | atexit 카운팅, 프롬프트 재작성 |
| v6 | 0 | 안정 — 기능 추가 | 성능 타이밍, thinking A/B |
| **v7** | **0** | 안정 — 최종 다듬기 | assert_total 분모 수정 |

v1에서 7개 태스크가 ALL_FAIL이었을 때, 저는 "역시 로컬 모델은 한계가 있구나"라고 생각했습니다.
틀렸습니다.
**7개 전부 하네스 버그였습니다.**

모델이 "실패"한다고 생각할 때마다, 실제 원인은 이런 것들이었습니다:
- 정답인데 포맷이 달라서 거부하는 assert 매칭
- 정확하지만 다른 접근법에 감점을 주는 스코어링 로직
- thinking이 긴 태스크에 대해 너무 짧은 타임아웃
- 저에게는 명확하지만 모델에게는 모호한 프롬프트

**하네스가 모델보다 버그가 더 많았습니다.**
벤치마크를 만드는 사람이라면 반드시 의심해야 합니다 — "모델이 틀린 건가, 내 평가가 틀린 건가?"

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

MAJORITY(다수결로 통과)인 4개 케이스에서 모델 간 실제 차이가 드러납니다:
- **DGX1 (Qwen 3.5)**: 상태 추적이 필요한 멀티턴 태스크에서 약합니다
- **B-Tree**: DGX1이 노드 분할(node splitting)에서 일관되게 실패합니다 — 구조적 한계입니다
- **Code Review**: 5090과 DGX2 모두 pickle 보안 검사를 놓칩니다 — 비결정적(non-deterministic) 문제입니다

---

## 배운 것들

### 1. 벤치마크 품질이 모델 품질보다 중요합니다

모델은 v1부터 문제가 없었습니다.
평가 프레임워크가 깨져 있었습니다.
벤치마크에서 모델이 분명히 풀 수 있는 문제를 "실패"한다고 나온다면, **버그는 벤치마크에 있습니다**.

### 2. 점수 ≠ 품질

100.0점은 "모든 구조적 검사를 통과했다"는 뜻이지, "좋은 코드를 생성했다"는 뜻이 아닙니다.
실제로 100.0점을 받았지만 미묘한 버그(nested validation edge case)가 있는 코드를 발견했습니다.
검증기는 설계된 것만 측정합니다.
여러분이 측정하고 싶은 것을 측정하는 게 아닙니다.

### 3. n=3은 필수입니다

n=1로는 모델 한계와 추론 비결정성(inference non-determinism)을 구분할 수 없습니다.
vLLM의 MTP(Multi-Token Prediction)는 네 번 중 한 번꼴로 완전히 다른 출력을 만듭니다.
n=3 평균으로 신호와 노이즈를 분리해야 합니다.

### 4. 35B의 적정 지점(sweet spot)

명확한 스펙이 있는 코딩 태스크에서, Qwen 35B MoE (3B active) FP8은 상용 도구와 품질이 동등합니다.

격차가 나타나는 영역:
- 모호한 다단계 태스크 (상용 도구의 instruction following이 우수)
- 롱 컨텍스트 검색 (로컬 128GB vs 클라우드 1M 토큰)
- 도구 사용 및 파일 시스템 연동 (Claude Code의 통합 레이어)

그 외 — 함수 작성, 디버깅, 알고리즘, 리팩토링 — 에서는 로컬 모델로 충분합니다.

---

## 결론: 로컬 LLM은 "아직 멀었다"가 아닙니다

많은 개발자가 "로컬 모델은 상용 서비스에 비해 한참 부족하다"고 생각합니다.
이 실험 결과는 다릅니다.

**명확한 명세의 코딩 태스크에서 로컬 35B는 Claude Code와 동등합니다.**
상용 도구의 진짜 가치는 코드 품질이 아니라, 통합 환경(tool use, 긴 컨텍스트, 파일 시스템 접근)에 있습니다.

그리고 무엇보다 — 벤치마크를 만들었다면, 모델을 의심하기 전에 벤치마크를 의심하십시오.

---

## 코드

세 하네스 모두 zero-dependency Python입니다:
- **cc-crosscheck**: `~/GitHub/cc-crosscheck/` — 254 tests, dual-35B architecture
- **dgx-duo**: `~/GitHub/dgx-duo/` — 9 modules, 254 tests, pair mode
- **Local-Trinity**: `~/GitHub/local-trinity/` — 17 modules, 290 tests, 3-node unified

문서 정리 후 공개 릴리스 예정입니다.

---

*개인 하드웨어에서의 개인 벤치마크입니다.
모델: Qwen 3.5/3.6 35B-A3B (FP8 및 Q4).
상용 비교 대상: Claude Code with Opus 4.6 (v2.1.109).
Anthropic, OpenAI, Alibaba와 제휴 관계 없습니다.*
