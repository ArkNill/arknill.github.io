---
title: "Claude Code의 Thinking Token은 쿼터에 포함된다 — 하지만 볼 수 없다"
date: 2026-04-06
description: "8,794건의 API 요청 프록시 데이터가 보여주는 것: 눈에 보이는 출력은 Claude Code 쿼터 소비의 절반도 설명하지 못한다. Extended thinking token이 보이지 않는 핵심 비용으로 추정된다."
tags: ["claude-code", "ai", "debugging", "open-source"]
showToc: true
TocOpen: false
draft: false
---

Claude Code Max 20 ($200/월)의 5시간 쿼터 전체를 사용해도 **눈에 보이는 출력 토큰은 0.9M~1.6M**에 불과하다. Opus 모델로 수 시간 작업한 결과치고는 현저히 낮은 수치다.

나머지는 어디로 갔는가? Extended thinking token은 API 응답의 `output_tokens` 필드에 **포함되지 않는다**. JSONL 세션 로그에서 볼 수 없고, 프록시를 통해 집계할 수도 없으며, Claude Code는 이를 모니터링하거나 제어할 방법을 제공하지 않는다. 그러나 쿼터는 thinking token이 출력 토큰 요율로 집계되는 것과 일치하는 속도로 소진된다.

이것은 추측이 아니다. 측정 방법은 다음과 같다.

## 사건의 시작

2026년 4월 1일, Max 20 구독이 정상적인 코딩 중 **70분 만에 100% 사용량**에 도달했다. JSONL 분석 결과 캐시 적중률이 **36.1%**로 나타났다 — 정상이라면 90% 이상이어야 한다. v2.1.89에서 v2.1.68로 다운그레이드하자 캐시가 즉시 **97.6%**로 회복되었다.

또 하나의 "저도요" 이슈를 제출하는 대신, 투명한 모니터링 프록시를 직접 만들었다.

## 측정 방법

[cc-relay](https://github.com/ArkNill/claude-code-hidden-problem-analysis)는 공식 `ANTHROPIC_BASE_URL` 환경 변수를 사용하여 Claude Code와 Anthropic API 사이에 위치한다. `anthropic-ratelimit-unified-*` 헤더를 포함한 모든 요청과 응답을 아무것도 수정하지 않고 로깅한다.

6일간: **8,794건의 요청** 로깅, **3,702건**에서 rate limit 헤더 캡처. 소스 감사 완료, 재현 가능.

## 데이터

프록시가 밝힌 서버 측 쿼터 아키텍처: **이중 슬라이딩 윈도우** — 5시간 카운터와 7일 카운터. 캡처된 요청의 100%에서 5시간 윈도우가 병목이었다.

**5시간 사용률 1%당** (Max 20, $200/월):

| 지표 | 범위 |
|------|------|
| 1%당 눈에 보이는 출력 | 9K~16K 토큰 |
| 1%당 캐시 읽기 | 1.5M~2.1M 토큰 |
| 1%당 전체 가시 토큰 | 1.6M~2.1M 토큰 |

1%당 가시 출력이 9K~16K이면, 5시간 윈도우 전체(100%)를 사용해도 출력 토큰은 0.9M~1.6M에 불과하다. 참고로, 이는 전체 작업 세션 동안 중간 크기 코드 생성 30~50회에 해당하는 분량이다.

가시 출력과 실제 쿼터 비용 사이의 격차는 **extended thinking token이 지배적 비용 요인**이면서 사용자에게 완전히 보이지 않는다는 것과 일치한다.

## 독립적 검증

커뮤니티 구성원들이 별도 조사를 통해 동일한 결론에 도달했다:

**[@fgrosswig](https://github.com/fgrosswig)**는 [64배 예산 감소 포렌식](https://github.com/anthropics/claude-code/issues/38335#issuecomment-4189537353)을 발표했다 — 18일간 듀얼 머신 JSONL 비교로 3월 26일: 3.2B 토큰 제한 없음 → 4월 5일: 90% 사용률에서 88M. 가시 토큰 수가 소비량을 설명하지 못한다.

**[@Commandershadow9](https://github.com/Commandershadow9)**는 [34~143배 용량 감소](https://github.com/anthropics/claude-code/issues/41506#issuecomment-4189508296)를 문서화하고, 캐시 수정이 효과가 있음을 확인했지만, 용량 감소가 클라이언트 측 버그와 무관함을 보여주었다. Thinking token이 독립적으로 원인으로 지목되었다.

4월 6일 주에 정확한 영향을 측정하기 위한 thinking 비활성화 격리 테스트가 계획되어 있다.

## 더 심각한 문제: 6개의 추가 버그

thinking token 사각지대만이 문제가 아니다. 조사 결과 **5개 계층에 걸친 7개의 확인된 버그**가 발견되었다:

### 수정됨 (v2.1.91)

| 버그 | 내용 | 영향 |
|------|------|------|
| **Sentinel** | 독립 실행형 바이너리가 캐시 접두사를 깨뜨림 | 캐시가 4~17%로 하락, 20배 비용 증가 ([#40524](https://github.com/anthropics/claude-code/issues/40524)) |
| **Resume** | `--resume`이 전체 컨텍스트를 캐시 없이 재생 | 500K 대화에서 resume당 ~$0.15 ([#34629](https://github.com/anthropics/claude-code/issues/34629)) |

**아직 업데이트하지 않았다면 v2.1.91로 업데이트하라.** 가장 큰 비용 유출이 수정된다.

### 미수정

| 버그 | 내용 | 영향 |
|------|------|------|
| **Microcompact** | 서버가 오래된 tool 결과를 조용히 제거 | 327건의 제거 이벤트 측정. 모델이 이전 컨텍스트를 잃음 ([#42542](https://github.com/anthropics/claude-code/issues/42542)) |
| **Budget cap** | tool 결과에 대한 200K 문자 집계 상한 | 파일 15~20개 읽은 후 오래된 결과가 1~41자로 잘림. 한 세션에서 261건 |
| **False rate limiter** | 클라이언트가 가짜 "Rate limit reached" 생성 | 65개 세션에서 151건의 합성 오류 — API 호출 0건 ([#40584](https://github.com/anthropics/claude-code/issues/40584)) |
| **JSONL inflation** | Extended thinking이 로그 항목을 중복 생성 | 로컬 집계에서 2.87배 토큰 인플레이션 ([#41346](https://github.com/anthropics/claude-code/issues/41346)) |

Microcompact와 budget cap이 특히 악질적이다: **서버 측 GrowthBook A/B 테스트 플래그**로 제어된다. Anthropic은 클라이언트 업데이트 없이 동작을 변경할 수 있으며, 환경 변수로 우회할 수 없다. 1M 컨텍스트 비용을 지불하는 사용자가 실질적으로 얻는 것은 내장 도구용 200K tool 결과 예산이다.

## Anthropic의 대응

4월 2일, Lydia Hallie (Anthropic, Product)가 [X에 게시](https://x.com/lydiahallie/status/2039800715607187906):

> *"피크 시간 제한이 더 빡빡해졌고 1M 컨텍스트 세션이 커졌습니다. 대부분은 이것 때문입니다. 중간에 몇 가지 버그를 수정했지만, 과다 청구한 것은 없습니다."*

데이터와 다른 부분:

- **"과다 청구한 것은 없다"** — Bug 5는 200K 이후 tool 결과를 1~41자로 잘라낸다. 261건 측정. 1M 컨텍스트 비용을 지불하는 사용자가 1M의 tool 결과를 받지 못한다.
- **"피크 시간 제한이 더 빡빡하다"** — 프록시는 시간대에 관계없이 3,702건 요청의 100%에서 `representative-claim` = `five_hour`를 보여준다. 주말 데이터도 동일 패턴.
- **Thinking token** — 언급 없음. 지배적 비용 요인이 사용자에게 보이지 않으며, 모니터링하거나 제어할 방법이 없다.

14개월에 걸친 **91건 이상의 GitHub 이슈**에서 Anthropic은 **GitHub에 공식 응답을 단 한 건도 게시하지 않았다**. 모든 소통은 개인 X 게시물과 changelog를 통해서만 이루어졌다.

## 할 수 있는 것

1. **v2.1.91로 업데이트** — 캐시 회귀 수정
2. **`--resume` 또는 `--continue` 사용 금지** — 새 세션을 시작하라
3. **주기적으로 새 세션 시작** — 200K budget cap이 오래된 tool 결과를 조용히 잘라낸다
4. **터미널 하나만 사용** — 다수의 터미널은 캐시를 공유하지 않는다

자가 진단 가이드: [09_QUICKSTART.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/09_QUICKSTART.md)

## 이것은 커뮤니티의 노력이었다

이 분석은 퍼즐의 각 조각을 독립적으로 발견한 17명의 커뮤니티 구성원의 작업 위에 구축되었다:

- [@Sn3th](https://github.com/Sn3th) — microcompact 메커니즘 (Bug 4), GrowthBook 플래그, budget 파이프라인 (Bug 5) 발견
- [@rwp65](https://github.com/rwp65) — 클라이언트 측 가짜 rate limiter (Bug 3) 발견, 상세 로그 증거
- [@fgrosswig](https://github.com/fgrosswig) — 듀얼 머신 18일 JSONL 비교로 64배 예산 감소 포렌식
- [@Commandershadow9](https://github.com/Commandershadow9) — 34~143배 용량 감소 분석 및 thinking token 가설
- [@kolkov](https://github.com/kolkov) — [ccdiag](https://github.com/kolkov/ccdiag) 제작, v2.1.91 `--resume` 회귀 3건 식별
- [@simpolism](https://github.com/simpolism) — resume 캐시 수정 패치 (99.7~99.9% 적중률)
- [@luongnv89](https://github.com/luongnv89) — 캐시 TTL 분석, [CUStats](https://custats.info) 및 [context-stats](https://github.com/luongnv89/cc-context-stats) 제작
- [@dancinlife](https://github.com/dancinlife) — organizationUuid 쿼터 풀링 및 oauthAccount 교차 오염 버그
- [@weilhalt](https://github.com/weilhalt) — rate-limit 헤더 모니터링용 [BudMon](https://github.com/weilhalt/budmon) 제작
- [@arizonawayfarer](https://github.com/arizonawayfarer) — Windows GrowthBook 플래그 덤프로 크로스 플랫폼 일관성 확인
- [@dbrunet73](https://github.com/dbrunet73) — 실제 OTel 비교 데이터 (v2.1.88 vs v2.1.90)로 캐시 개선 확인
- [@maiarowsky](https://github.com/maiarowsky) — v2.1.90에서 Bug 3 확인, 13개 세션에서 합성 항목 26건
- [@edimuj](https://github.com/edimuj) — grep/파일 읽기 토큰 낭비 측정 (3.5M 토큰 / 1,800+건), [tokenlean](https://github.com/edimuj/tokenlean) 제작
- [@amicicixp](https://github.com/amicicixp) — v2.1.90 캐시 개선을 전후 비교 테스트로 검증
- [@pablofuenzalidadf](https://github.com/pablofuenzalidadf) — 구 Docker 버전 비용 유출 보고 — 핵심 서버 측 증거 ([#37394](https://github.com/anthropics/claude-code/issues/37394))
- [@SC7639](https://github.com/SC7639) — 3월 중순 타임라인 확인하는 추가 회귀 데이터
- Reddit 커뮤니티 — 캐시 sentinel 메커니즘의 [리버스 엔지니어링 분석](https://www.reddit.com/r/ClaudeAI/s/AY2GHQa5Z6)

## 전체 분석

**[github.com/ArkNill/claude-code-hidden-problem-analysis](https://github.com/ArkNill/claude-code-hidden-problem-analysis)**

11개 문서: 버그 기술 상세, 프록시 캡처 rate limit 헤더, JSONL 포렌식, 벤치마크, 91건 이상 이슈의 14개월 타임라인, 커뮤니티 도구. 26 commits, 전체 데이터 교차 검증 완료.

---

*커뮤니티 연구 및 개인 측정. Anthropic의 보증 없음. 모든 모니터링은 공식 도구와 문서화된 기능을 사용.*
