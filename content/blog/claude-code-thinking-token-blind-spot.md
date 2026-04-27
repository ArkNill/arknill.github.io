---
title: "I Tracked 42,363 Claude Code API Calls. Here's Where Your Quota Actually Goes."
date: 2026-04-06
lastmod: 2026-04-27
description: "19 days of transparent proxy data on Claude Code Max 20 — token breakdown, 11 bugs found, Opus 4.7 impact, and Anthropic's April 23 postmortem. Independent datasets from other researchers confirmed the pattern and corrected my original hypothesis."
tags: ["claude-code", "ai", "debugging", "open-source"]
showToc: true
TocOpen: false
draft: false
---

I pay $200/month for Claude Code Max 20. On April 1, my quota hit **100% in 70 minutes** during normal coding. That turned out to be two cache bugs — Anthropic fixed them in v2.1.90–91, and it made a real difference.

But even after the fix, I wanted to understand where the quota actually goes. So I filed issues, dug into community threads, and built a transparent proxy to measure every API call.

After the cache bugs were fixed, I pinned **v2.1.91** and kept measuring. Later I pinned **v2.1.109** — the last stable release before Opus 4.7. Over 19 days, I logged **42,363 API calls** across 298 sessions — all on cache-fixed, pinned versions.

Here's what my data shows.

---

## Where the Tokens Go

The token volume breakdown across 42,363 proxy-captured requests:

| Category | Tokens | Share |
|----------|--------|-------|
| Cache Read | 4,613,123,498 | **97.0%** |
| Cache Creation | 91,391,928 | 1.9% |
| Input | 33,337,097 | 0.7% |
| Output | 20,559,648 | **0.4%** |

Cache read is not waste — it's how the API works. Every request re-reads the conversation context from cache, which is more efficient than rebuilding it from scratch. **The question is whether this volume counts against your quota, and at what rate.** More on that below.

Each 1% of my 5-hour quota window produced **9,000–16,000 tokens** of visible output. A full 100% window means 0.9M–1.6M tokens of actual code across an entire session.

## My Original Hypothesis Was Wrong

I initially suspected extended thinking tokens were the hidden cost — they don't appear in `output_tokens`, can't be seen in logs, can't be counted through a proxy. It seemed like the obvious explanation for where the quota was going.

I was wrong. An independent researcher proved it (see [Independent Corroboration](#independent-corroboration) below).

## What My Proxy Found: 11 Bugs Across 5 Layers

Beyond the token breakdown, the proxy and JSONL analysis uncovered **11 confirmed bugs**. Anthropic fixed the two worst — **B1 Sentinel** (cache prefix corruption) and **B2 Resume** (full context replay) — in v2.1.90–91. All numbers below are from data collected *after* those fixes.

The remaining 8 are different in nature — context management, tool result handling, local logging — and have survived **20 releases** (v2.1.92–v2.1.112):

### The ones that hurt most

**B5 — Tool result truncation.** My proxy logged **167,770 truncation events**. Tool results are silently capped at 200K aggregate characters — anything older gets cut to 1–41 chars. You're paying for 1M context, but tool results get a 200K budget. This is controlled by a server-side setting, not client code.

**B3 — Fake rate limits.** The *client* generates "Rate limit reached" errors without making an API call. I found 151 synthetic errors across 65 sessions. You're being throttled by your own tool. ([#40584](https://github.com/anthropics/claude-code/issues/40584))

**B4 — Silent context removal.** Old tool results are silently stripped from context. 5,437 removal events measured. The model loses earlier context without warning. ([#42542](https://github.com/anthropics/claude-code/issues/42542))

**B10 — Context injection.** Deprecated TaskOutput messages inject up to 87K tokens into context, triggering cascading autocompact. Last verified on v2.1.109.

### Others documented

B8 (log inflation, 2.37x duplication across 532 files), B8a (JSONL corruption from concurrent tools), B9 (/branch message duplication, 6%→73% context), B11 (zero reasoning tokens — Anthropic acknowledged on HN). Full evidence: [01_BUGS.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/01_BUGS.md)

> *Bug status was last verified on v2.1.109–112. Some may have been addressed in newer releases — check the linked issues for current status.*

## Opus 4.7: What I Measured vs. What Others Found

Opus 4.7 launched April 15. I documented the tokenizer and search regressions; two other researchers independently measured the quota burn from their own accounts:

**What I measured:**
- **Tokenizer +35%**: the same content consumes more tokens on 4.7
- **Long-context search regression**: accuracy dropped from 91.9% to 59.2% overall, and from 78.3% to 32.2% at 524K–1024K context (source: [Opus 4.7 advisory](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/16_OPUS-47-ADVISORY.md))
- **Model pin bypass**: v2.1.111+ ignores `settings.json` model setting and silently switches to 4.7 ([#49503](https://github.com/anthropics/claude-code/issues/49503))

**What others measured independently (their data, their accounts):**

| Who | Plan / Region | Finding |
|-----|--------------|---------|
| [@cnighswonger](https://github.com/cnighswonger) | Max 5x, US, 71 calls | Sustained burn **2.4x** vs 4.6. After Anthropic expanded rate limits post-launch, per-call cost improved ~4.2x. |
| [@fgrosswig](https://github.com/fgrosswig) | Max 5x, EU, A/B test | **12.5x** sustained on simultaneous 4.6/4.7 test. Cold start (first few calls only): up to 50x. |

> *Each row is a different person, different account, different region. The range (2.4x–12.5x sustained) reflects real variance across conditions. The 50x is a cold-start outlier, not typical usage.*

**Update (April 23):** Anthropic restored default effort to `high` for Pro/Max in v2.1.117 and reverted the "≤25 words" system prompt in v2.1.116. The model pin bypass (#49503) and long-context search regression remain open as of April 27.

**Recommendation**: Stay on v2.1.109 with `/model claude-opus-4-6` until the model pin bypass and search regression are resolved.

## Different Plans Behave Differently

My proxy captured data on two tiers (Max 20x and Max 5x, same account holder):

| Tier | Haiku calls | Opus calls |
|------|------------|------------|
| Max 20x | **20.77%** | 78.84% |
| Max 5x | **0.11%** | 83.46% |

The **190x difference** in Haiku calls is largely architectural — Claude Code's built-in **Explore subagent uses Haiku by default** (confirmed by [cnighswonger's subagent transcript analysis](https://github.com/fgrosswig/claude-gateway/issues/1)). Max 20x usage patterns may trigger more subagent calls. This is a structural difference, not necessarily model substitution.

Separately, fgrosswig observed 14% Haiku model substitution on a Pro-tier account (EU), while cnighswonger saw zero mismatches on Max 5x (US) across 14K+ calls. Whether this is tier-dependent, session-length-dependent, or load-dependent remains an open question with data from only two accounts.

## Anthropic's Response

**Update (April 23):** Anthropic published a [postmortem](https://www.anthropic.com/engineering/april-23-postmortem) acknowledging **three product-layer bugs** that degraded Claude Code between March and April 2026:

1. **Effort downgrade** — Default effort level was silently changed from `high` to `medium` on March 4 (v2.1.68). Pro and Max users ran at reduced quality for **48 days** until restored on April 21 (v2.1.117).
2. **Thinking cache pattern clear** — A change on March 26 (v2.1.85) broke thinking token caching, fixed April 10 (v2.1.101). **Not recorded in CHANGELOG.**
3. **"≤25 words" system prompt** — Added April 16 (v2.1.111), caused 3% coding quality drop, reverted April 20 (v2.1.116). **Not recorded in CHANGELOG.**

The postmortem confirms none of these involved model weight changes — they were configuration and prompt-layer issues. Anthropic also reset usage meters and launched [@ClaudeDevs](https://x.com/ClaudeDevs) for future incident communication.

**What remains unaddressed:** The specific bugs documented in this investigation (B3–B11) — tool result truncation (167,770 events), fake rate limits (151 events), silent context removal (5,437 events) — are not covered by the postmortem. The quota formula question (0x→1x cache_read weight hypothesis) also remains unanswered.

**Prior communication (April 2):** Before the postmortem, Lydia Hallie (Anthropic, Product) [posted on X](https://x.com/lydiahallie/status/2039800715607187906):

> *"We fixed a few bugs along the way, but none were over-charging you."*

The cache fixes (B1/B2) she referenced were real and helpful. The three bugs acknowledged in the postmortem came after this statement.

## What You Can Do

1. **Pin v2.1.109** — last version before Opus 4.7 model pin bypass
2. **Pin your model**: `/model claude-opus-4-6` at session start
3. **Start fresh sessions** — don't use `--resume` or `--continue`
4. **Rotate sessions** — the 200K budget cap silently truncates older tool results
5. **One terminal only** — multiple terminals don't share cache

Self-diagnosis guide: [09_QUICKSTART.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/09_QUICKSTART.md)

---

## Independent Corroboration

After I published the original version of this analysis, several independent researchers brought their own data. Their findings both confirmed and corrected mine.

### seanGSISG: 178K calls that changed my conclusion

[@seanGSISG](https://github.com/seanGSISG) contributed **178,009 API calls** from a separate Max 20x account spanning December 2025 through April 2026 — the 5 months of "before-data" my proxy couldn't capture. ([Full analysis with 6 scripts](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/3))

Their JSONL logs contain actual content blocks, allowing direct measurement of thinking tokens via character-based heuristics. **Thinking tokens account for an estimated 0.0–0.1% of total quota** — not the primary cause I originally hypothesized.

Instead, their data is consistent with a **quota formula change**: modeling cache_read weight at 0x (old) vs 1x (new) produces a 10–15x multiplier that matches observed behavior. Under the 0x model, zero days exceeded the budget in their entire 5-month dataset. Under the 1x model, 18 days exceeded it.

Our per-1% measurements converge independently:

| Metric | My data (42K calls, Apr) | seanGSISG (178K calls, Dec–Apr) |
|--------|--------------------------|--------------------------------|
| CacheRead per 1% | 1.5M–2.1M | 1.62M–1.72M |
| cache_read share | 96–99% | 89.8–95.2% |

> *Anthropic has not confirmed or denied a quota formula change. The 0x→1x model is seanGSISG's best-fit hypothesis based on observed data — not a confirmed fact. The slight gap in cache_read share reflects different measurement approaches (active-window per-1% vs monthly averages including cold starts).*

### Other independent datasets

- **[@cnighswonger](https://github.com/cnighswonger)** — 14K+ calls with an in-process interceptor ([claude-code-cache-fix](https://github.com/cnighswonger/claude-code-cache-fix)). Measured Opus 4.7 burn, confirmed Explore subagent = Haiku by default, compared 4 agents across 21 days.
- **[@fgrosswig](https://github.com/fgrosswig)** — 18-day gateway forensics ([claude-gateway](https://github.com/fgrosswig/claude-gateway)). Ran simultaneous Opus 4.6/4.7 A/B test on the same account.
- **[@wpank](https://github.com/wpank)** — 47,810 requests, $10,700 total spend across version comparisons.
- **[@edimuj](https://github.com/edimuj)** — 3.5M tokens measuring token waste. Built [tokenlean](https://github.com/edimuj/tokenlean).

19 more contributors discovered bugs, built tools, and verified findings — credited in the [full contributor list](https://github.com/ArkNill/claude-code-hidden-problem-analysis#contributors).

> *Each dataset was collected independently — different people, machines, subscription plans (Max 20x / Max 5x / Pro), and regions. Numbers are never aggregated across datasets.*

## Full Data

Everything is open. Reproduce it yourself.

- **Analysis repository**: [github.com/ArkNill/claude-code-hidden-problem-analysis](https://github.com/ArkNill/claude-code-hidden-problem-analysis)
- **Consolidated dataset**: [DATASET.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/DATASET.md) — four independent datasets with methodology notes and cross-comparison constraints
- **Independent corroboration**: [Issue #3](https://github.com/ArkNill/claude-code-hidden-problem-analysis/issues/3) — seanGSISG's 178K-call analysis
- **Proxy tool**: [llm-relay](https://github.com/QuartzUnit/llm-relay) — `pip install llm-relay`

---

*My research, supported by independent contributors. Not affiliated with or endorsed by Anthropic. All monitoring uses the official `ANTHROPIC_BASE_URL` proxy mechanism.*
