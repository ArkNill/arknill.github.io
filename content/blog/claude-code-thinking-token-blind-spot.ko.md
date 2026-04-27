---
title: "Claude Code API 호출 42,363건을 추적했습니다. 쿼터가 실제로 어디에 쓰이는지 밝힙니다."
date: 2026-04-06
lastmod: 2026-04-27
description: "Claude Code Max 20에서 19일간의 투명 프록시 데이터 — 토큰 분석, 11개 버그 발견, Opus 4.7 영향, Anthropic의 4월 23일 포스트모템. 다른 연구자들의 독립 데이터셋이 패턴을 확인하고 저의 원래 가설을 교정했습니다."
tags: ["claude-code", "ai", "debugging", "open-source"]
showToc: true
TocOpen: false
draft: false
---

저는 Claude Code Max 20에 월 $200를 지불합니다.
4월 1일, 일반적인 코딩 중에 쿼터가 **70분 만에 100%**에 도달했습니다.
원인은 두 가지 캐시 버그였으며 — Anthropic이 v2.1.90–91에서 수정했고,
실제로 효과가 있었습니다.

하지만 수정 이후에도, 쿼터가 실제로 어디에 사용되는지 이해하고 싶었습니다.
그래서 이슈를 제출하고, 커뮤니티 스레드를 파헤치고,
모든 API 호출을 측정하기 위한 투명 프록시를 직접 구축했습니다.

캐시 버그가 수정된 후, **v2.1.91**을 고정하고 측정을 계속했습니다.
이후 **v2.1.109** — Opus 4.7 이전 마지막 안정 릴리스 — 로 고정했습니다.
19일에 걸쳐 298개 세션에서 **42,363건의 API 호출**을 로깅했습니다 —
모두 캐시 수정 완료된 고정 버전에서 수행되었습니다.

이것이 데이터가 보여주는 내용입니다.

---

## 토큰의 행방

42,363건의 프록시 캡처 요청에서 토큰 사용량 분석입니다:

| Category | Tokens | Share |
|----------|--------|-------|
| Cache Read | 4,613,123,498 | **97.0%** |
| Cache Creation | 91,391,928 | 1.9% |
| Input | 33,337,097 | 0.7% |
| Output | 20,559,648 | **0.4%** |

Cache read는 낭비가 아닙니다 — API가 작동하는 방식입니다.
매 요청마다 대화 컨텍스트를 캐시에서 다시 읽으며,
이는 처음부터 다시 구축하는 것보다 효율적입니다.
**문제는 이 용량이 쿼터에 포함되는지, 그리고 어떤 비율로 계산되는지입니다.**
이에 대해서는 아래에서 더 다룹니다.

5시간 쿼터 윈도우의 1%마다 **9,000–16,000 토큰**의 가시적 출력이 생성되었습니다.
윈도우 전체 100%는 전체 세션에 걸쳐 0.9M–1.6M 토큰의 실제 코드를 의미합니다.

## 원래 가설은 틀렸습니다

처음에 저는 extended thinking token이 숨겨진 비용이라고 의심했습니다 —
`output_tokens`에 나타나지 않고, 로그에서 볼 수 없으며,
프록시를 통해 집계할 수 없습니다.
쿼터가 소진되는 이유에 대한 명백한 설명처럼 보였습니다.

틀렸습니다.
독립 연구자가 이를 증명했습니다 (아래 [독립적 검증](#독립적-검증) 참조).

## 프록시가 발견한 것: 5개 계층에 걸친 11개 버그

토큰 분석 외에도, 프록시와 JSONL 분석으로 **11개의 확인된 버그**가 발견되었습니다.
Anthropic은 가장 심각한 두 가지 — **B1 Sentinel** (캐시 접두사 손상)과 **B2 Resume** (전체 컨텍스트 재전송) — 를 v2.1.90–91에서 수정했습니다.
아래의 모든 수치는 해당 수정 *이후* 수집된 데이터입니다.

나머지 8개는 성격이 다릅니다 —
컨텍스트 관리, tool result 처리, 로컬 로깅 —
그리고 **20개 릴리스** (v2.1.92–v2.1.112)에 걸쳐 생존했습니다:

### 가장 영향이 큰 버그들

**B5 — Tool result 잘림.** 프록시가 **167,770건의 잘림 이벤트**를 로깅했습니다.
Tool result는 집계 200K 문자에서 조용히 잘립니다 —
더 오래된 것은 1–41자로 축소됩니다.
1M 컨텍스트 비용을 지불하지만, tool result는 200K 예산을 받습니다.
이는 클라이언트 코드가 아닌 서버 측 설정으로 제어됩니다.

**B3 — 가짜 rate limit.** *클라이언트*가 API 호출 없이 "Rate limit reached" 오류를 생성합니다.
65개 세션에서 151건의 합성 오류를 발견했습니다.
자신의 도구에 의해 쓰로틀링되는 것입니다. ([#40584](https://github.com/anthropics/claude-code/issues/40584))

**B4 — 무음 컨텍스트 제거.** 오래된 tool result가 컨텍스트에서 조용히 제거됩니다.
5,437건의 제거 이벤트가 측정되었습니다.
모델이 경고 없이 이전 컨텍스트를 잃습니다. ([#42542](https://github.com/anthropics/claude-code/issues/42542))

**B10 — 컨텍스트 주입.** 폐기된 TaskOutput 메시지가 최대 87K 토큰을 컨텍스트에 주입하여
연쇄적 autocompact를 유발합니다.
v2.1.109에서 마지막으로 확인되었습니다.

### 기타 문서화된 버그

B8 (로그 인플레이션, 532개 파일에서 2.37x 중복),
B8a (동시 도구로 인한 JSONL 손상),
B9 (/branch 메시지 중복, 6%→73% 컨텍스트),
B11 (reasoning 토큰 0건 — Anthropic이 HN에서 인정).
전체 증거: [01_BUGS.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/01_BUGS.md)

> *버그 상태는 v2.1.109–112에서 마지막으로 확인되었습니다. 일부는 최신 릴리스에서 수정되었을 수 있습니다 — 현재 상태는 링크된 이슈를 확인하십시오.*

## Opus 4.7: 직접 측정한 것 vs. 다른 사람들이 발견한 것

Opus 4.7은 4월 15일에 출시되었습니다.
저는 tokenizer와 검색 회귀를 문서화했으며;
두 명의 다른 연구자가 각자의 계정에서 쿼터 소모를 독립적으로 측정했습니다:

**직접 측정한 내용:**
- **Tokenizer +35%**: 동일한 콘텐츠가 4.7에서 더 많은 토큰을 소비합니다
- **Long-context 검색 회귀**: 정확도가 전체 91.9%에서 59.2%로, 524K–1024K 컨텍스트에서는 78.3%에서 32.2%로 하락했습니다 (출처: [Opus 4.7 advisory](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/16_OPUS-47-ADVISORY.md))
- **Model pin 우회**: v2.1.111+는 `settings.json` 모델 설정을 무시하고 조용히 4.7로 전환합니다 ([#49503](https://github.com/anthropics/claude-code/issues/49503))

**다른 사람들이 독립적으로 측정한 내용 (각자의 데이터, 각자의 계정):**

| Who | Plan / Region | Finding |
|-----|--------------|---------|
| [@cnighswonger](https://github.com/cnighswonger) | Max 5x, US, 71 calls | 지속적 소모 4.6 대비 **2.4x**. Anthropic이 출시 후 rate limit를 확장한 후, 호출당 비용은 ~4.2x 개선. |
| [@fgrosswig](https://github.com/fgrosswig) | Max 5x, EU, A/B test | 동시 4.6/4.7 테스트에서 지속적 **12.5x**. Cold start (처음 몇 호출만): 최대 50x. |

> *각 행은 서로 다른 사람, 다른 계정, 다른 지역입니다. 범위 (2.4x–12.5x sustained)는 조건에 따른 실제 분산을 반영합니다. 50x는 cold-start 아웃라이어이며, 일반적인 사용 패턴이 아닙니다.*

**업데이트 (4월 23일):** Anthropic은 v2.1.117에서 Pro/Max에 대해 기본 effort를 `high`로 복원했고
v2.1.116에서 "≤25 words" system prompt를 되돌렸습니다.
Model pin 우회 (#49503)와 long-context 검색 회귀는 4월 27일 기준 여전히 미해결입니다.

**권장 사항**: model pin 우회와 검색 회귀가 해결될 때까지
v2.1.109에 `/model claude-opus-4-6`을 사용하십시오.

## 플랜에 따라 동작이 다릅니다

프록시가 두 개 티어 (Max 20x와 Max 5x, 동일 계정 소유자)의 데이터를 캡처했습니다:

| Tier | Haiku calls | Opus calls |
|------|------------|------------|
| Max 20x | **20.77%** | 78.84% |
| Max 5x | **0.11%** | 83.46% |

Haiku 호출의 **190배 차이**는 대부분 구조적인 것입니다 —
Claude Code의 내장 **Explore subagent가 기본적으로 Haiku를 사용합니다**
([cnighswonger의 subagent transcript 분석](https://github.com/fgrosswig/claude-gateway/issues/1)에서 확인).
Max 20x 사용 패턴이 더 많은 subagent 호출을 유발할 수 있습니다.
이는 구조적 차이이며, 반드시 모델 대체를 의미하지는 않습니다.

별도로, fgrosswig는 Pro 티어 계정(EU)에서 14% Haiku 모델 대체를 관찰했으며,
cnighswonger는 Max 5x(US)에서 14K+ 호출에 걸쳐 불일치를 전혀 발견하지 못했습니다.
이것이 티어 의존적인지, 세션 길이 의존적인지, 부하 의존적인지는
두 계정의 데이터만으로는 열린 질문으로 남습니다.

## Anthropic의 대응

**업데이트 (4월 23일):** Anthropic은 [포스트모템](https://www.anthropic.com/engineering/april-23-postmortem)을 발표하여
2026년 3월부터 4월 사이 Claude Code를 저하시킨 **세 가지 프로덕트 레이어 버그**를 인정했습니다:

1. **Effort 하향 조정** — 기본 effort 수준이 3월 4일(v2.1.68)에 `high`에서 `medium`으로 조용히 변경되었습니다. Pro와 Max 사용자는 4월 21일(v2.1.117) 복원까지 **48일간** 저하된 품질로 작업했습니다.
2. **Thinking cache pattern clear** — 3월 26일(v2.1.85)의 변경이 thinking token 캐싱을 깨뜨렸으며, 4월 10일(v2.1.101)에 수정되었습니다. **CHANGELOG에 기록되지 않았습니다.**
3. **"≤25 words" system prompt** — 4월 16일(v2.1.111)에 추가되어 3% 코딩 품질 저하를 야기했으며, 4월 20일(v2.1.116)에 되돌려졌습니다. **CHANGELOG에 기록되지 않았습니다.**

포스트모템은 이들 중 모델 가중치 변경과 관련된 것은 없다고 확인합니다 —
모두 설정 및 프롬프트 레이어 문제였습니다.
Anthropic은 또한 사용량 미터를 리셋하고 향후 인시던트 소통을 위해 [@ClaudeDevs](https://x.com/ClaudeDevs)를 개설했습니다.

**미해결 사항:** 이 조사에서 문서화된 특정 버그들(B3–B11) —
tool result 잘림(167,770건), 가짜 rate limit(151건),
무음 컨텍스트 제거(5,437건) — 는 포스트모템에서 다루지 않습니다.
쿼터 공식 질문(0x→1x cache_read 가중치 가설)도 미답변으로 남아 있습니다.

**이전 소통 (4월 2일):** 포스트모템 이전에, Lydia Hallie (Anthropic, Product)가 [X에 게시](https://x.com/lydiahallie/status/2039800715607187906)했습니다:

> *"We fixed a few bugs along the way, but none were over-charging you."*

그녀가 언급한 캐시 수정(B1/B2)은 실제로 도움이 되었습니다.
포스트모템에서 인정된 세 가지 버그는 이 발언 이후에 나왔습니다.

## 할 수 있는 것

1. **v2.1.109 고정** — Opus 4.7 model pin 우회 이전 마지막 버전
2. **모델 고정**: 세션 시작 시 `/model claude-opus-4-6`
3. **새 세션으로 시작** — `--resume` 또는 `--continue` 사용 금지
4. **세션 교체** — 200K budget cap이 오래된 tool result를 조용히 잘라냅니다
5. **터미널 하나만 사용** — 다수의 터미널은 캐시를 공유하지 않습니다

자가 진단 가이드: [09_QUICKSTART.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/09_QUICKSTART.md)

---

## 독립적 검증

이 분석의 원래 버전을 공개한 후,
여러 독립 연구자들이 자신의 데이터를 가지고 왔습니다.
그들의 발견은 저의 분석을 확인하기도 하고 교정하기도 했습니다.

### seanGSISG: 결론을 바꾼 178K 호출

[@seanGSISG](https://github.com/seanGSISG)는 별도의 Max 20x 계정에서
2025년 12월부터 2026년 4월까지 **178,009건의 API 호출**을 기여했습니다 —
저의 프록시가 캡처할 수 없었던 5개월간의 "이전 데이터"입니다.
([6개 스크립트가 포함된 전체 분석](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/3))

해당 JSONL 로그에는 실제 content block이 포함되어 있어,
문자 기반 휴리스틱으로 thinking token을 직접 측정할 수 있습니다.
**Thinking token은 전체 쿼터의 약 0.0–0.1%를 차지하는 것으로 추정됩니다** —
제가 원래 가설로 세웠던 주요 원인이 아닙니다.

대신, 그들의 데이터는 **쿼터 공식 변경**과 일치합니다:
cache_read 가중치를 0x(이전) vs 1x(이후)로 모델링하면
관찰된 동작과 일치하는 10–15x 승수가 나옵니다.
0x 모델에서는 5개월 전체 데이터에서 예산을 초과한 날이 0일이었습니다.
1x 모델에서는 18일이 초과했습니다.

1%당 측정값이 독립적으로 수렴합니다:

| Metric | My data (42K calls, Apr) | seanGSISG (178K calls, Dec–Apr) |
|--------|--------------------------|--------------------------------|
| CacheRead per 1% | 1.5M–2.1M | 1.62M–1.72M |
| cache_read share | 96–99% | 89.8–95.2% |

> *Anthropic은 쿼터 공식 변경을 확인하거나 부인하지 않았습니다. 0x→1x 모델은 관찰된 데이터에 기반한 seanGSISG의 최적 적합 가설이며 — 확인된 사실이 아닙니다. cache_read share의 약간의 차이는 서로 다른 측정 방식(활성 윈도우 1%당 vs cold start를 포함한 월간 평균)을 반영합니다.*

### 기타 독립 데이터셋

- **[@cnighswonger](https://github.com/cnighswonger)** — in-process interceptor ([claude-code-cache-fix](https://github.com/cnighswonger/claude-code-cache-fix))를 사용한 14K+ 호출. Opus 4.7 소모를 측정하고, Explore subagent = Haiku 기본값을 확인하며, 21일에 걸쳐 4개 에이전트를 비교.
- **[@fgrosswig](https://github.com/fgrosswig)** — 18일 gateway 포렌식 ([claude-gateway](https://github.com/fgrosswig/claude-gateway)). 동일 계정에서 Opus 4.6/4.7 동시 A/B 테스트 수행.
- **[@wpank](https://github.com/wpank)** — 47,810건의 요청, 버전 비교에 걸친 총 $10,700 지출.
- **[@edimuj](https://github.com/edimuj)** — 토큰 낭비 측정에 3.5M 토큰. [tokenlean](https://github.com/edimuj/tokenlean) 제작.

19명의 추가 기여자들이 버그를 발견하고, 도구를 만들고, 발견을 검증했습니다 —
[전체 기여자 목록](https://github.com/ArkNill/claude-code-hidden-problem-analysis#contributors)에 기재되어 있습니다.

> *각 데이터셋은 독립적으로 수집되었습니다 — 서로 다른 사람, 머신, 구독 플랜(Max 20x / Max 5x / Pro), 지역. 데이터셋 간 수치를 합산하지 않았습니다.*

## 전체 데이터

모든 것이 공개되어 있습니다.
직접 재현해 보십시오.

- **분석 레포지토리**: [github.com/ArkNill/claude-code-hidden-problem-analysis](https://github.com/ArkNill/claude-code-hidden-problem-analysis)
- **통합 데이터셋**: [DATASET.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/DATASET.md) — 방법론 노트와 교차 비교 제약 조건이 포함된 4개의 독립 데이터셋
- **독립적 검증**: [Issue #3](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/3) — seanGSISG의 178K 호출 분석
- **프록시 도구**: [llm-relay](https://github.com/QuartzUnit/llm-relay) — `pip install llm-relay`

---

*독립 기여자들의 지원을 받은 개인 연구입니다.
Anthropic과 제휴하거나 보증받지 않습니다.
모든 모니터링은 공식 `ANTHROPIC_BASE_URL` 프록시 메커니즘을 사용합니다.*
