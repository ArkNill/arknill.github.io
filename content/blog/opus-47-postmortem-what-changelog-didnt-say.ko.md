---
title: "Opus 4.7 포스트모템: CHANGELOG가 말하지 않은 것"
date: 2026-04-27
description: "Anthropic은 48일간 Claude Code를 저하시킨 세 가지 제품 계층 버그를 인정했습니다. 포스트모템을 CHANGELOG와 대조 검증하면 구조적 투명성 격차가 드러납니다 — 3건 중 2건은 문서화가 전무합니다. 그리고 포스트모템 범위를 넘어서는 5건의 새로운 이슈가 여전히 존재합니다."
tags: ["claude-code", "ai", "postmortem", "transparency"]
showToc: true
TocOpen: false
draft: false
---

4월 23일, Anthropic은 3월 4일부터 4월 20일까지 Claude Code를 저하시킨 세 가지 제품 계층 버그를 인정하는 [포스트모템](https://www.anthropic.com/engineering/april-23-postmortem)을 발표했습니다.
모델 가중치 변경은 없었습니다 — 세 가지 이슈 모두 하네스/제품 계층에서 발생한 것입니다.

저는 모든 주장을 공개 CHANGELOG(3,285줄, v2.1.68–v2.1.119),
`gh issue view`를 통한 8건의 GitHub 이슈, 그리고 10개의 외부 소스와 대조 검증했습니다.
**36건 검증 — 28건 확인, 5건 부분 확인, 3건 미의존.**

포스트모템이 말하는 것, CHANGELOG가 실제로 보여주는 것,
그리고 아직 수정되지 않은 것을 정리합니다.

---

## 인정된 세 가지 버그

### 1. Effort 다운그레이드 (3월 4일 – 4월 21일)

3월 4일, Claude Code의 기본 추론 effort가 `high`에서 `medium`으로 변경되었습니다.
CHANGELOG(v2.1.68)는 이를 제품 개선으로 표현했습니다:

> *"Opus 4.6 now defaults to medium effort for Max and Team subscribers. Medium effort works well for most tasks — it's the sweet spot between speed and thoroughness."*

월 $100–$200를 지불하는 Pro 및 Max 구독자는 **48일간** 저하된 품질로 운영되었습니다.
수정은 두 단계로 배포되었습니다:

| 날짜 | 버전 | 범위 |
|------|------|------|
| 4월 7일 | v2.1.94 | API-key, Bedrock/Vertex, Team, Enterprise |
| **4월 21일** | **v2.1.117** | **Pro/Max 구독자** |

가장 높은 요금을 지불하는 티어가 마지막으로 수정되었습니다.
CHANGELOG에 "revert"라는 단어는 한 번도 등장하지 않습니다 — 도입과 수정 모두 전향적 제품 언어를 사용합니다.

### 2. Thinking Cache 버그 (3월 26일 – 4월 10일)

캐싱 최적화가 유휴 세션에서 오래된 thinking 블록을 정리하도록 설계되었습니다.
그러나 해당 플래그는 **모든 후속 턴에서** 발동되어,
매 API 호출마다 이전 추론을 파괴했습니다.
Claude가 건망증이 있고 반복적으로 보이는 증상이 나타났습니다.

- **도입:** 3월 26일 (v2.1.85)
- **수정:** 4월 10일 (v2.1.101)
- **CHANGELOG 기재:** 없음. 버그의 도입과 수정 모두 공개 문서화가 없습니다.

"thinking cache," "cache clear," "thinking prun"이라는 용어는 3,285줄 CHANGELOG **어디에도** 등장하지 않습니다.

### 3. "≤25 words" 시스템 프롬프트 (4월 16일 – 4월 20일)

Opus 4.7 출시와 같은 날, 시스템 프롬프트가 추가되었습니다:

> *"Length limits: keep text between tool calls to <=25 words. Keep final responses to <=100 words unless the task requires more detail."*

이로 인해 Opus 4.6과 4.7 모두에서 코딩 품질이 **3%** 저하되었으며 —
Anthropic 자체 평가로 검증되었습니다.

- **도입:** 4월 16일 (v2.1.111)
- **롤백:** 4월 20일 (v2.1.116)
- **CHANGELOG 기재:** 없음. 도입과 롤백 모두 문서화되지 않았습니다.

---

## 투명성 패턴

| 버그 | CHANGELOG 도입 기재 | CHANGELOG 수정 기재 | 포스트모템 |
|------|---------------------|---------------------|------------|
| Effort 다운그레이드 | 있음 ("sweet spot") | 있음 ("Changed") | "Wrong tradeoff" |
| Thinking cache | **없음** | **없음** | 인정 |
| ≤25 words 프롬프트 | **없음** | **없음** | 인정 |

**인정된 3건의 버그 중 2건은 CHANGELOG 문서화가 전무합니다.**
문서화된 1건은 프로모션 언어를 사용했으며, 회귀로 인정되지 않았습니다.
포스트모템이 이러한 변경이 저하를 야기했다는 최초의 공개 인정이었습니다.

시스템 프롬프트 변경은 구조적으로 보이지 않습니다 —
모든 대화 전에 주입되는 지시사항을 수정하지만 공개 CHANGELOG에는 추적되지 않습니다.
사용자는 버전 간 시스템 프롬프트를 diff할 수 없습니다.

---

## 포스트-포스트모템: 지속되는 5건의 이슈 (v2.1.117–119)

포스트모템은 세 가지 이슈 모두 4월 20일부로 해결되었다고 명시합니다.
그러나 4월 22–24일에 제출된 이슈들은 포스트모템 범위를 넘어서는 문제를 보여줍니다:

### Subagent model pin 무시 ([#52502](https://github.com/anthropics/claude-code/issues/52502))

Agent frontmatter의 `model: haiku`가 무시되어 — 모든 작업이 Opus에서 실행됩니다.
한 사용자의 `/usage`는 Haiku $0.0005 대비 Opus $10.87을 보여줍니다.
비용 최적화된 멀티 에이전트 워크플로우를 설계한 사용자들이 자신도 모르게 모든 것을 고가 모델에서 실행하고 있습니다.

### Effort override 우회 ([#52534](https://github.com/anthropics/claude-code/issues/52534))

`CLAUDE_CODE_EFFORT_LEVEL` 환경변수와 `settings.json effortLevel`이 UI 레벨 플래그(`unpinOpus47LaunchEffort`)에 의해 덮어씌워집니다.
이 플래그는 사용자가 대화형으로 `/effort`를 사용해야만 해제됩니다 — 닭과 달걀 문제입니다.
자동화된 워크플로우에서는 비용을 제어할 수 없습니다.

### Auto-compact 임계값 변경 ([#52522](https://github.com/anthropics/claude-code/issues/52522))

v2.1.117은 auto-compact 임계값을 ~200K에서 ~1M 토큰으로 변경했습니다.
CHANGELOG는 이를 "Fix"라고 부릅니다 (200K 대신 Opus 4.7의 네이티브 1M에 대해 계산하도록).
그러나 200K 체제에서 운영하던 사용자에게는 **하룻밤 사이 5배 토큰 사용량**이 발생했습니다.
동작 변경은 문서화되었지만, 비용 영향은 문서화되지 않았습니다.

### Self-conversation 안전 이슈 ([#52228](https://github.com/anthropics/claude-code/issues/52228))

모델이 보관된 문서에서 "Human:" 프롬프트를 조작하여 워크스테이션에서 일방적 행동과 함께 자기 대화를 시작했습니다.
안전 관련 실패입니다 — 모델이 가공의 사용자 입력을 생성하고 이를 행동에 대한 승인으로 사용했습니다.

### CLAUDE.md 규칙 위반 ([#52652](https://github.com/anthropics/claude-code/issues/52652))

"NEVER execute Git commands" 규칙의 명시적 위반 — 무허가 `git stash && ... && git stash pop` 실행.
모델은 실행 후 결과를 검증하지 않았습니다.

**5건 모두에 대한 Anthropic의 응답: 4월 27일 기준 없음.**

---

## 미해결 사항

포스트모템은 세 가지 특정 하네스 버그를 설명합니다.
다음 사항은 다루지 않습니다:

- **B5** — Tool result 잘림 (프록시 데이터에서 167,770건의 이벤트)
- **B3** — 클라이언트 측 거짓 속도 제한 (151건의 합성 오류)
- **B4** — 무음 컨텍스트 제거 (5,437건의 이벤트)
- **#49302** — Cache 계량 이상 (7배 과청구 보고)
- **#49503** — Model pin 우회
- **Long-context 회귀** — 256K에서 91.9% → 59.2%, 1M에서 78.3% → 32.2% (Anthropic 자체 시스템 카드)

이들은 전체 증거 추적과 함께 [16_OPUS-47-ADVISORY.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/16_OPUS-47-ADVISORY.md)에 문서화되어 있습니다.

---

## 권장 사항

**v2.1.109가 여전히 안전한 버전입니다.**
Opus 4.7 model pin 우회, 토크나이저 변경(+35%)보다 이전 버전이며,
세 가지 포스트모템 버그는 모두 v2.1.101/v2.1.116에서 이미 수정되었습니다.
단, v2.1.109 + Opus 4.6 조합은 포스트-포스트모템 이슈를 완전히 회피합니다.

Opus 4.6은 최소 **2027년 2월 5일**까지 활성 상태입니다 — 약 10개월의 여유가 있습니다.
고정 버전에서 안정적으로 운영 중이라면, 업그레이드할 긴급성은 없습니다.

---

## 전체 증거

- **포스트모템 대조 검증 (36건):** [17_OPUS-47-POSTMORTEM-ANALYSIS.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/17_OPUS-47-POSTMORTEM-ANALYSIS.md)
- **Opus 4.7 기술 권고문:** [16_OPUS-47-ADVISORY.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/16_OPUS-47-ADVISORY.md)
- **버그 증거:** [01_BUGS.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/01_BUGS.md)
- **프록시 도구:** [llm-relay](https://github.com/QuartzUnit/llm-relay) — `pip install llm-relay`

---

*독립적 분석입니다. Anthropic과 제휴하거나 보증을 받지 않았습니다.*
