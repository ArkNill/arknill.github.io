---
title: "Claude Code API 42,363건을 전수 추적해봤습니다 — 쿼터는 대체 어디로 가는 걸까요"
date: 2026-04-06
lastmod: 2026-04-27
description: "Claude Code Max 20 구독자가 19일간 투명 프록시로 전수 측정한 토큰 소비 구조, 11개 버그, Opus 4.7 영향, 그리고 Anthropic 4월 23일 포스트모템까지. 독립 연구자 4명의 데이터가 원래 가설을 교정했습니다."
tags: ["claude-code", "ai", "debugging", "open-source"]
showToc: true
TocOpen: false
draft: false
---

## 70분 만에 쿼터 100% — 시작은 여기였습니다

Claude Code Max 20은 월 $200 구독 요금제로, Anthropic이 "최고 사용량 사용자"를 위해 만든 플랜입니다.
5시간 윈도우 단위로 쿼터가 리셋되는 구조인데, 4월 1일 평범한 코딩 중에 **70분 만에 100%**를 찍었습니다.

원인은 두 가지 캐시 버그였고, Anthropic이 v2.1.90~91에서 수정한 뒤 실제로 개선되었습니다.
하지만 "수정 이후에도 쿼터가 체감보다 빨리 소진된다"는 느낌이 남았습니다.

그래서 직접 측정하기로 했습니다.
이슈를 올리고, 커뮤니티 스레드를 파헤치고, 모든 API 호출을 기록하는 투명 프록시를 만들었습니다.

캐시 버그 수정 이후 **v2.1.91**을 고정(pin)하고 측정을 시작했습니다.
이후 Opus 4.7 출시 직전의 마지막 안정 버전인 **v2.1.109**로 고정해 계속했습니다.
19일간 298개 세션에서 **42,363건의 API 호출**을 로깅한 결과를 공유합니다.

---

## 원래 가설: "Thinking token이 범인일 것이다"

처음에는 extended thinking token(모델이 답변을 생성하기 전 내부적으로 추론하는 토큰)이 숨겨진 비용이라고 생각했습니다.
이유는 명확했습니다:

- `output_tokens` 필드에 포함되지 않습니다
- 로그에서 볼 수 없습니다
- 프록시를 통해서도 집계가 불가능합니다

쿼터 소진의 명백한 원인처럼 보였습니다.

하지만 결론부터 말하면, **이 가설은 틀렸습니다.**
독립 연구자가 178K건의 실데이터로 증명했습니다 (아래 [독립 검증](#독립-검증) 참조).

---

## 토큰은 실제로 어디에 쓰이는가

42,363건 전수 캡처 결과입니다:

| Category | Tokens | Share |
|----------|--------|-------|
| Cache Read | 4,613,123,498 | **97.0%** |
| Cache Creation | 91,391,928 | 1.9% |
| Input | 33,337,097 | 0.7% |
| Output | 20,559,648 | **0.4%** |

Cache read(캐시 읽기)는 낭비가 아닙니다.
Claude Code는 매 요청마다 대화 컨텍스트를 캐시에서 다시 읽는 구조이고, 처음부터 재구축하는 것보다 효율적입니다.

**핵심 질문은 이겁니다: 이 cache read 볼륨이 쿼터에 포함되는지, 그리고 어떤 가중치로 계산되는지.**
이 부분은 독립 검증 섹션에서 다시 다룹니다.

체감 수치로 말하면, 5시간 쿼터 윈도우의 1%마다 **9,000~16,000 토큰**의 가시적 출력이 생성되었습니다.
전체 100%를 쓰면 한 세션에서 약 0.9M~1.6M 토큰의 실제 코드가 나옵니다.

---

## 프록시가 발견한 11개 버그

토큰 분석 외에도 프록시와 JSONL 분석으로 **11개의 확인된 버그**를 발견했습니다.
Anthropic은 가장 심각한 두 건 — **B1 Sentinel**(캐시 접두사 손상)과 **B2 Resume**(전체 컨텍스트 재전송) — 을 v2.1.90~91에서 수정했습니다.
아래 수치는 모두 해당 수정 **이후** 수집된 데이터입니다.

나머지 8건은 성격이 다릅니다.
컨텍스트 관리, tool result 처리, 로컬 로깅 관련이고, **20개 릴리스**(v2.1.92~v2.1.112)를 거치면서도 생존했습니다.

### 영향이 큰 버그들

**B5 — Tool result 잘림.**
프록시가 **167,770건의 잘림 이벤트**를 기록했습니다.
Tool result(도구 실행 결과)는 집계 200K 문자에서 조용히 잘립니다.
더 오래된 결과는 1~41자로 축소됩니다.
1M 컨텍스트 비용을 내지만, tool result에는 200K 예산만 배정되는 셈입니다.

**B3 — 가짜 rate limit.**
*클라이언트*가 API 호출 없이 "Rate limit reached" 에러를 자체 생성합니다.
65개 세션에서 151건의 합성 에러를 발견했습니다.
자기 도구에 의해 쓰로틀링되는 것입니다. ([#40584](https://github.com/anthropics/claude-code/issues/40584))

**B4 — 무음 컨텍스트 제거.**
오래된 tool result가 컨텍스트에서 조용히 삭제됩니다.
5,437건의 제거 이벤트를 측정했습니다.
모델이 경고 없이 이전 맥락을 잃습니다. ([#42542](https://github.com/anthropics/claude-code/issues/42542))

**B10 — 컨텍스트 주입.**
폐기된 TaskOutput 메시지가 최대 87K 토큰을 컨텍스트에 주입하여 연쇄적 autocompact를 유발합니다.
v2.1.109에서 마지막 확인되었습니다.

### 기타 문서화된 버그

B8 (로그 인플레이션, 532개 파일에서 2.37x 중복),
B8a (동시 도구 실행으로 인한 JSONL 손상),
B9 (/branch 메시지 중복으로 컨텍스트 6%에서 73%까지 팽창),
B11 (reasoning 토큰 0건 — Anthropic이 HN에서 인정).
전체 증거: [01_BUGS.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/01_BUGS.md)

> *버그 상태는 v2.1.109~112에서 마지막으로 확인되었습니다. 일부는 최신 릴리스에서 수정되었을 수 있으니, 현재 상태는 링크된 이슈를 확인하시기 바랍니다.*

---

## Opus 4.7의 영향

Opus 4.7은 4월 15일 출시되었습니다.
직접 tokenizer와 검색 회귀를 측정했고, 두 명의 독립 연구자가 각자 계정에서 쿼터 소모를 측정했습니다.

### 직접 측정한 내용

- **Tokenizer +35%**: 동일한 콘텐츠가 4.7에서 35% 더 많은 토큰을 소비합니다
- **Long-context 검색 회귀**: 정확도가 전체 91.9%에서 59.2%로, 524K~1024K 컨텍스트에서는 78.3%에서 32.2%로 하락했습니다 (출처: [Opus 4.7 advisory](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/16_OPUS-47-ADVISORY.md))
- **Model pin 우회**: v2.1.111 이상에서 `settings.json`의 모델 설정을 무시하고 조용히 4.7로 전환합니다 ([#49503](https://github.com/anthropics/claude-code/issues/49503))

### 타 연구자 독립 측정 (각자의 데이터, 각자의 계정)

| Who | Plan / Region | Finding |
|-----|--------------|---------|
| [@cnighswonger](https://github.com/cnighswonger) | Max 5x, US, 71 calls | 지속 소모 4.6 대비 **2.4x**. Anthropic이 출시 후 rate limit 확장 뒤 호출당 비용 ~4.2x 개선. |
| [@fgrosswig](https://github.com/fgrosswig) | Max 5x, EU, A/B test | 동시 4.6/4.7 테스트에서 지속 **12.5x**. Cold start(처음 몇 호출): 최대 50x. |

> *각 행은 서로 다른 사람, 다른 계정, 다른 지역입니다. 범위(2.4x~12.5x sustained)는 조건에 따른 실제 분산을 반영합니다. 50x는 cold-start 아웃라이어이며 일반적 패턴이 아닙니다.*

**업데이트 (4월 23일):** Anthropic은 v2.1.117에서 Pro/Max 기본 effort를 `high`로 복원했고, v2.1.116에서 "25 words" system prompt를 되돌렸습니다.
Model pin 우회(#49503)와 long-context 검색 회귀는 4월 27일 기준 여전히 미해결입니다.

**권장 사항**: model pin 우회와 검색 회귀가 해결될 때까지 v2.1.109에 `/model claude-opus-4-6`을 사용하십시오.

---

## 플랜에 따라 동작이 다릅니다

프록시가 두 개 티어(Max 20x와 Max 5x, 동일 계정 소유자)의 데이터를 캡처했습니다:

| Tier | Haiku calls | Opus calls |
|------|------------|------------|
| Max 20x | **20.77%** | 78.84% |
| Max 5x | **0.11%** | 83.46% |

Haiku 호출의 **190배 차이**는 대부분 구조적입니다.
Claude Code의 내장 **Explore subagent가 기본적으로 Haiku를 사용**하기 때문입니다
([cnighswonger의 subagent transcript 분석](https://github.com/fgrosswig/claude-gateway/issues/1)에서 확인).
Max 20x 사용 패턴이 더 많은 subagent 호출을 유발하는 것으로 보입니다.
이는 구조적 차이이며, 반드시 "모델 대체"를 의미하지는 않습니다.

별도로, fgrosswig는 Pro 티어 계정(EU)에서 14% Haiku 모델 대체를 관찰했고, cnighswonger는 Max 5x(US)에서 14K+ 호출 중 불일치를 전혀 발견하지 못했습니다.
티어 의존인지, 세션 길이 의존인지, 서버 부하 의존인지는 두 계정 데이터만으로 결론 내릴 수 없습니다.

---

## Anthropic의 대응

**4월 23일**, Anthropic이 [포스트모템](https://www.anthropic.com/engineering/april-23-postmortem)을 발표하여 2026년 3~4월 Claude Code를 저하시킨 **세 가지 프로덕트 레이어 버그**를 인정했습니다:

1. **Effort 하향 조정** — 3월 4일(v2.1.68)에 기본 effort가 `high`에서 `medium`으로 조용히 변경되었습니다. Pro와 Max 사용자가 **48일간** 저하된 품질로 작업했고, 4월 21일(v2.1.117)에야 복원되었습니다.
2. **Thinking cache pattern clear** — 3월 26일(v2.1.85) 변경이 thinking token 캐싱을 깨뜨렸고, 4월 10일(v2.1.101)에 수정되었습니다. **CHANGELOG에 기록되지 않았습니다.**
3. **"25 words" system prompt** — 4월 16일(v2.1.111)에 추가되어 3% 코딩 품질 저하를 야기했고, 4월 20일(v2.1.116)에 되돌려졌습니다. **CHANGELOG에 기록되지 않았습니다.**

포스트모템은 이들 모두 모델 가중치 변경이 아닌, 설정 및 프롬프트 레이어 문제라고 확인합니다.
Anthropic은 사용량 미터를 리셋하고, 향후 인시던트 소통을 위해 [@ClaudeDevs](https://x.com/ClaudeDevs)를 개설했습니다.

**그런데 미해결인 것들이 있습니다.**
이 조사에서 문서화된 버그(B3~B11) — tool result 잘림(167,770건), 가짜 rate limit(151건), 무음 컨텍스트 제거(5,437건) — 는 포스트모템에서 다루지 않습니다.
쿼터 공식 질문(0x에서 1x로의 cache_read 가중치 변경 가설)도 미답변입니다.

**이전 소통 (4월 2일):** 포스트모템 이전에 Lydia Hallie(Anthropic, Product)가 [X에 게시](https://x.com/lydiahallie/status/2039800715607187906)했습니다:

> *"We fixed a few bugs along the way, but none were over-charging you."*

당시 언급한 캐시 수정(B1/B2)은 실제로 도움이 되었습니다.
하지만 포스트모템에서 인정된 세 가지 버그는 이 발언 이후에 나온 것들입니다.

---

## 독립 검증

분석 원본을 공개한 뒤, 여러 독립 연구자가 자체 데이터를 가져왔습니다.
이들의 발견이 제 분석을 확인하기도 하고, 교정하기도 했습니다.

### seanGSISG: 결론을 바꾼 178K건

[@seanGSISG](https://github.com/seanGSISG)가 별도의 Max 20x 계정에서 2025년 12월부터 2026년 4월까지 **178,009건의 API 호출** 데이터를 기여했습니다.
제 프록시가 캡처할 수 없었던 5개월간의 "이전 데이터"입니다.
([6개 스크립트 포함 전체 분석](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/3))

해당 JSONL 로그에는 실제 content block이 포함되어 있어, 문자 기반 휴리스틱으로 thinking token을 직접 측정할 수 있었습니다.
결과: **thinking token은 전체 쿼터의 약 0.0~0.1%를 차지** — 제가 원래 세웠던 가설의 주요 원인이 아니었습니다.

대신, 이 데이터는 **쿼터 공식 변경**과 일치합니다.
cache_read 가중치를 0x(이전)에서 1x(이후)로 모델링하면, 관찰된 행동과 맞는 10~15x 승수가 나옵니다.
0x 모델에서는 5개월 전체 데이터 중 예산 초과일이 **0일**이었습니다.
1x 모델에서는 **18일**이 초과했습니다.

두 데이터셋의 1%당 측정값이 독립적으로 수렴합니다:

| Metric | My data (42K calls, Apr) | seanGSISG (178K calls, Dec–Apr) |
|--------|--------------------------|--------------------------------|
| CacheRead per 1% | 1.5M–2.1M | 1.62M–1.72M |
| cache_read share | 96–99% | 89.8–95.2% |

> *Anthropic은 쿼터 공식 변경을 확인하거나 부인하지 않았습니다. 0x에서 1x 모델은 관찰 데이터에 기반한 seanGSISG의 최적 적합 가설이며, 확인된 사실이 아닙니다. cache_read share 차이는 서로 다른 측정 방식(활성 윈도우 1%당 vs cold start 포함 월간 평균)을 반영합니다.*

### 기타 독립 데이터셋

- **[@cnighswonger](https://github.com/cnighswonger)** — in-process interceptor([claude-code-cache-fix](https://github.com/cnighswonger/claude-code-cache-fix))를 사용한 14K+ 호출. Opus 4.7 소모 측정, Explore subagent = Haiku 기본값 확인, 21일간 4개 에이전트 비교.
- **[@fgrosswig](https://github.com/fgrosswig)** — 18일 gateway 포렌식([claude-gateway](https://github.com/fgrosswig/claude-gateway)). 동일 계정에서 Opus 4.6/4.7 동시 A/B 테스트.
- **[@wpank](https://github.com/wpank)** — 47,810건 요청, 버전 비교 총 $10,700 지출.
- **[@edimuj](https://github.com/edimuj)** — 토큰 낭비 측정에 3.5M 토큰 투입. [tokenlean](https://github.com/edimuj/tokenlean) 제작.

19명의 추가 기여자가 버그를 발견하고, 도구를 만들고, 발견을 검증했습니다 — [전체 기여자 목록](https://github.com/ArkNill/claude-code-hidden-problem-analysis#contributors).

> *각 데이터셋은 독립적으로 수집되었습니다 — 서로 다른 사람, 머신, 구독 플랜(Max 20x / Max 5x / Pro), 지역. 데이터셋 간 수치는 합산하지 않았습니다.*

---

## 지금 할 수 있는 것

1. **v2.1.109 고정** — Opus 4.7 model pin 우회 이전 마지막 버전
2. **모델 고정**: 세션 시작 시 `/model claude-opus-4-6`
3. **새 세션으로 시작** — `--resume`이나 `--continue` 사용을 피합니다
4. **세션을 주기적으로 교체** — 200K budget cap이 오래된 tool result를 조용히 잘라냅니다
5. **터미널 하나만 사용** — 다수의 터미널은 캐시를 공유하지 않습니다

자가 진단 가이드: [09_QUICKSTART.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/09_QUICKSTART.md)

---

## 전체 데이터

모든 것이 공개되어 있습니다.
직접 재현해 보십시오.

- **분석 레포지토리**: [github.com/ArkNill/claude-code-hidden-problem-analysis](https://github.com/ArkNill/claude-code-hidden-problem-analysis)
- **통합 데이터셋**: [DATASET.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/DATASET.md) — 방법론 노트와 교차 비교 제약 조건이 포함된 4개 독립 데이터셋
- **독립 검증**: [Issue #3](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/3) — seanGSISG의 178K건 분석
- **프록시 도구**: [llm-relay](https://github.com/QuartzUnit/llm-relay) — `pip install llm-relay`

---

*독립 기여자들의 지원을 받은 개인 연구입니다.
Anthropic과 제휴하거나 보증받지 않습니다.
모든 모니터링은 공식 `ANTHROPIC_BASE_URL` 프록시 메커니즘을 사용합니다.*
