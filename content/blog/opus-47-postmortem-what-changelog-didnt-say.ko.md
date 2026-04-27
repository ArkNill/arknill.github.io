---
title: "Opus 4.7 포스트모템: CHANGELOG가 말하지 않은 것"
date: 2026-04-28
description: "Anthropic이 48일간 Claude Code를 저하시킨 제품 계층 버그 3건을 인정했습니다. 포스트모템을 CHANGELOG와 대조하면 구조적 투명성 격차가 드러납니다 — 3건 중 2건은 문서화 자체가 없었습니다. 그리고 포스트모템 범위 밖에서 5건의 이슈가 여전히 진행 중입니다."
tags: ["claude-code", "ai", "postmortem", "transparency"]
showToc: true
TocOpen: false
draft: true
---

## 배경

소프트웨어 회사가 장애를 공식 인정하는 문서를 postmortem(포스트모템)이라고 합니다.
"무엇이 잘못되었고, 왜 발생했으며, 어떻게 재발을 막을 것인가"를 공개하는 투명성 관행입니다.

그리고 CHANGELOG는 버전별 변경사항을 사용자에게 알리는 공식 기록입니다.
사용자가 "이번 업데이트에서 무엇이 바뀌었는지" 확인할 수 있는 유일한 통로이기도 합니다.

4월 23일, Anthropic이 [포스트모템](https://www.anthropic.com/engineering/april-23-postmortem)을 발행했습니다.
3월 4일부터 4월 20일까지, Claude Code에서 제품 계층(harness/product layer) 버그 3건이 성능을 저하시켰다는 내용입니다.
모델 가중치 변경은 없었습니다 — 세 건 모두 모델을 감싸는 제품 레이어의 문제였습니다.

---

## 검증 방법

저는 포스트모템의 모든 주장을 공개 CHANGELOG(3,285줄, v2.1.68–v2.1.119)와 대조했습니다.
`gh issue view`로 GitHub 이슈 8건을 확인하고, 외부 소스 10건을 교차 참조했습니다.

**36건의 주장을 검증한 결과: 28건 확인, 5건 부분 확인, 3건 의존하지 않음.**

아래에서 포스트모템이 무엇을 말했는지, CHANGELOG에는 실제로 무엇이 기록되어 있는지, 그리고 아직 해결되지 않은 것이 무엇인지를 정리합니다.

---

## 인정된 버그 3건

### 1. Effort 다운그레이드 (3월 4일 – 4월 21일)

3월 4일, Claude Code의 기본 추론 effort가 `high`에서 `medium`으로 변경되었습니다.
effort란 모델이 응답을 생성할 때 얼마나 깊이 사고하는지를 제어하는 설정입니다.

CHANGELOG(v2.1.68)는 이 변경을 제품 개선으로 표현했습니다:

> *"Opus 4.6 now defaults to medium effort for Max and Team subscribers. Medium effort works well for most tasks — it's the sweet spot between speed and thoroughness."*

그런데 실제로는 월 $100–$200를 지불하는 Pro/Max 구독자가 **48일간** 저하된 품질로 서비스를 이용한 것입니다.
수정은 두 단계로 진행되었습니다:

| Date | Version | Scope |
|------|---------|-------|
| April 7 | v2.1.94 | API-key, Bedrock/Vertex, Team, Enterprise |
| **April 21** | **v2.1.117** | **Pro/Max subscribers** |

가장 높은 요금을 내는 티어가 가장 마지막에 수정되었습니다.
CHANGELOG에서 "revert"라는 단어는 한 번도 등장하지 않습니다.
도입과 수정 모두 전향적 제품 언어(forward-looking product language)로만 기술되어 있습니다.

### 2. Thinking Cache 버그 (3월 26일 – 4월 10일)

캐싱 최적화가 유휴 세션에서 오래된 thinking block을 정리하도록 설계되었습니다.
하지만 실제로는 해당 플래그가 **매 턴마다** 작동하여, 매 API 호출 시 이전 추론을 삭제했습니다.
결과적으로 Claude가 반복적이고 건망증이 있는 것처럼 보였습니다.

- **도입:** 3월 26일 (v2.1.85)
- **수정:** 4월 10일 (v2.1.101)
- **CHANGELOG 기록:** 없음. 버그 도입과 수정 모두 공개 문서화되지 않았습니다.

"thinking cache," "cache clear," "thinking prun"이라는 용어는 3,285줄의 CHANGELOG **어디에도** 등장하지 않습니다.

### 3. "≤25 words" 시스템 프롬프트 (4월 16일 – 4월 20일)

Opus 4.7 출시일에 시스템 프롬프트가 추가되었습니다:

> *"Length limits: keep text between tool calls to <=25 words. Keep final responses to <=100 words unless the task requires more detail."*

시스템 프롬프트(system prompt)란 사용자 대화 이전에 모델에 주입되는 지시문입니다.
사용자에게는 보이지 않지만 모델의 행동을 근본적으로 제어합니다.

이 변경은 Opus 4.6과 4.7 모두에서 코딩 품질을 **3%** 저하시켰습니다.
Anthropic 자체 평가(evaluation)로 확인된 수치입니다.

- **도입:** 4월 16일 (v2.1.111)
- **복원:** 4월 20일 (v2.1.116)
- **CHANGELOG 기록:** 없음. 도입과 복원 모두 문서화되지 않았습니다.

---

## 투명성 패턴

| Bug | CHANGELOG Introduction | CHANGELOG Fix | Postmortem |
|-----|----------------------|---------------|------------|
| Effort downgrade | Present ("sweet spot") | Present ("Changed") | "Wrong tradeoff" |
| Thinking cache | **Absent** | **Absent** | Admitted |
| ≤25 words prompt | **Absent** | **Absent** | Admitted |

**인정된 3건 중 2건은 CHANGELOG 기록이 전무합니다.**
유일하게 기록된 1건은 프로모션 언어로 포장되었을 뿐, 퇴행(regression)으로 인정된 적이 없습니다.
포스트모템이 이 변경들이 저하를 유발했음을 공개적으로 인정한 최초의 문서입니다.

시스템 프롬프트 변경은 구조적으로 불가시합니다.
모든 대화 이전에 주입되는 지시문을 수정하면서도 공개 CHANGELOG에는 추적되지 않습니다.
즉, 사용자가 버전 간 시스템 프롬프트를 diff로 비교할 방법이 없습니다.

---

## 포스트모템 이후: 여전히 존재하는 5건의 이슈 (v2.1.117–119)

포스트모템은 4월 20일 기준 세 건 모두 해결되었다고 명시합니다.
하지만 4월 22–24일에 보고된 이슈들은 포스트모템의 범위를 넘어서는 문제를 보여줍니다.

### Subagent 모델 pin 무시 ([#52502](https://github.com/anthropics/claude-code/issues/52502))

에이전트 frontmatter에서 `model: haiku`를 지정해도 무시됩니다.
모든 작업이 Opus로 실행됩니다.
한 사용자의 `/usage` 기록은 Haiku $0.0005 대비 Opus $10.87을 보여줍니다.
비용 최적화를 위해 멀티 에이전트 워크플로우를 설계한 사용자가, 자신도 모르게 전부 비싼 모델로 실행하고 있는 상황입니다.

### Effort override 우회 ([#52534](https://github.com/anthropics/claude-code/issues/52534))

`CLAUDE_CODE_EFFORT_LEVEL` 환경변수와 `settings.json effortLevel`이 UI 레벨 플래그(`unpinOpus47LaunchEffort`)에 의해 무시됩니다.
이 플래그는 사용자가 대화형으로 `/effort`를 사용해야만 해제됩니다 — 전형적인 닭과 달걀 문제입니다.
결국 자동화된 워크플로우에서는 비용을 제어할 수 없습니다.

### Auto-compact 임계값 변경 ([#52522](https://github.com/anthropics/claude-code/issues/52522))

v2.1.117이 auto-compact 임계값을 ~200K에서 ~1M 토큰으로 변경했습니다.
CHANGELOG는 이를 "Fix"로 기술합니다(200K 대신 Opus 4.7의 네이티브 1M에 맞춰 계산하도록 수정).
하지만 200K 체제에서 운영하던 사용자에게는 **하룻밤 사이에 토큰 사용량이 5배** 증가한 것입니다.
동작 변경은 문서화되었으나, 비용 영향은 문서화되지 않았습니다.

### Self-conversation 안전 이슈 ([#52228](https://github.com/anthropics/claude-code/issues/52228))

모델이 아카이브된 문서에서 "Human:" 프롬프트를 생성하고 자체 대화를 시작한 후 워크스테이션에서 일방적 행동을 취했습니다.
안전성 측면의 실패입니다 — 모델이 가상의 사용자 입력을 생성하고 이를 행동의 근거로 사용한 것입니다.

### CLAUDE.md 규칙 위반 ([#52652](https://github.com/anthropics/claude-code/issues/52652))

"NEVER execute Git commands" 규칙을 명시적으로 위반하여 무단으로 `git stash && ... && git stash pop`을 실행했습니다.
모델은 실행 결과를 확인하지도 않았습니다.

**5건 모두에 대한 Anthropic의 응답: 4월 27일 현재 없음.**

---

## 여전히 미해결인 것들

포스트모템은 세 가지 특정 harness 버그를 설명합니다.
하지만 다음 사항에 대해서는 언급하지 않습니다:

- **B5** — Tool result truncation (제 프록시 데이터 기준 167,770건)
- **B3** — 클라이언트측 거짓 rate limiting (합성 오류 151건)
- **B4** — 무음 컨텍스트 제거 (5,437건)
- **#49302** — Cache 미터링 이상 (7배 과금 보고)
- **#49503** — Model pin bypass
- **Long-context regression** — 91.9% → 59.2% (256K), 78.3% → 32.2% (1M) (Anthropic 자체 system card 데이터)

이들은 [16_OPUS-47-ADVISORY.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/16_OPUS-47-ADVISORY.md)에 전체 증거와 함께 정리되어 있습니다.

---

## 권장 사항

**v2.1.109가 현재 안전한 버전입니다.**
이 버전은 Opus 4.7 model pin bypass, 토크나이저 변경(+35%), 그리고 포스트모템의 세 버그가 모두 수정된 이후(v2.1.101/v2.1.116) 시점에 해당합니다.
v2.1.109 + Opus 4.6 조합이면 포스트모템 이후의 이슈들을 전부 회피할 수 있습니다.

Opus 4.6은 최소 **2027년 2월 5일**까지 유지됩니다 — 약 10개월의 여유가 있습니다.
고정 버전에서 안정적으로 운영 중이라면, 급하게 업그레이드할 이유가 없습니다.

---

## 전체 증거

- **포스트모템 크로스체크 (36건):** [17_OPUS-47-POSTMORTEM-ANALYSIS.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/17_OPUS-47-POSTMORTEM-ANALYSIS.md)
- **Opus 4.7 기술 권고:** [16_OPUS-47-ADVISORY.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/16_OPUS-47-ADVISORY.md)
- **버그 증거:** [01_BUGS.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/01_BUGS.md)
- **프록시 도구:** [llm-relay](https://github.com/QuartzUnit/llm-relay) — `pip install llm-relay`

---

*독립 분석입니다. Anthropic과 제휴하거나 보증을 받지 않았습니다.*
