---
title: "Opus 4.7 Postmortem: What the Changelog Didn't Say"
date: 2026-04-27
description: "Anthropic admitted three product-layer bugs that degraded Claude Code for 48 days. Cross-checking the postmortem against the CHANGELOG reveals a structural transparency gap — 2 of 3 bugs had zero documentation. And 5 new issues persist beyond the postmortem's scope."
tags: ["claude-code", "ai", "postmortem", "transparency"]
showToc: true
TocOpen: false
draft: false
---

On April 23, Anthropic published a [postmortem](https://www.anthropic.com/engineering/april-23-postmortem) acknowledging three product-layer bugs that degraded Claude Code from March 4 through April 20.
No model weights were changed — all three issues were in the harness/product layer.

I cross-checked every claim against the public CHANGELOG (3,285 lines, v2.1.68–v2.1.119),
8 GitHub issues via `gh issue view`, and 10 external sources.
**36 claims checked — 28 confirmed, 5 partially confirmed, 3 not relied upon.**

Here's what the postmortem says, what the CHANGELOG actually shows,
and what still isn't fixed.

---

## The Three Admitted Bugs

### 1. Effort Downgrade (March 4 – April 21)

On March 4, Claude Code's default reasoning effort was changed from `high` to `medium`.
The CHANGELOG (v2.1.68) framed it as a product improvement:

> *"Opus 4.6 now defaults to medium effort for Max and Team subscribers. Medium effort works well for most tasks — it's the sweet spot between speed and thoroughness."*

Pro and Max subscribers — paying $100–$200/month — ran at reduced quality for **48 days**.
The fix was rolled out in two phases:

| Date | Version | Scope |
|------|---------|-------|
| April 7 | v2.1.94 | API-key, Bedrock/Vertex, Team, Enterprise |
| **April 21** | **v2.1.117** | **Pro/Max subscribers** |

The highest-paying tier was fixed last.
The word "revert" never appears in the CHANGELOG — both the introduction and the fix use forward-looking product language.

### 2. Thinking Cache Bug (March 26 – April 10)

A caching optimization was supposed to clear old thinking blocks from idle sessions.
Instead, the flag fired **on every subsequent turn**,
destroying prior reasoning on each API call.
Claude appeared forgetful and repetitive.

- **Introduced:** March 26 (v2.1.85)
- **Fixed:** April 10 (v2.1.101)
- **CHANGELOG entries:** Zero. The bug was introduced and fixed with no public documentation.

The terms "thinking cache," "cache clear," and "thinking prun" appear **nowhere** in the 3,285-line CHANGELOG.

### 3. "≤25 words" System Prompt (April 16 – April 20)

On the same day Opus 4.7 launched, a system prompt was added:

> *"Length limits: keep text between tool calls to <=25 words. Keep final responses to <=100 words unless the task requires more detail."*

This degraded coding quality by **3%** across both Opus 4.6 and 4.7 —
verified by Anthropic's own evaluations.

- **Introduced:** April 16 (v2.1.111)
- **Reverted:** April 20 (v2.1.116)
- **CHANGELOG entries:** Zero. Both introduction and revert undocumented.

---

## The Transparency Pattern

| Bug | CHANGELOG Introduction | CHANGELOG Fix | Postmortem |
|-----|----------------------|---------------|------------|
| Effort downgrade | Present ("sweet spot") | Present ("Changed") | "Wrong tradeoff" |
| Thinking cache | **Absent** | **Absent** | Admitted |
| ≤25 words prompt | **Absent** | **Absent** | Admitted |

**2 of 3 admitted bugs have zero CHANGELOG documentation.**
The one that was documented used promotional language, never acknowledged as a regression.
The postmortem was the first public admission that these changes caused degradation.

System prompt changes are structurally invisible —
they modify instructions injected before every conversation but are never tracked in the public CHANGELOG.
Users cannot diff system prompts between versions.

---

## Post-Postmortem: 5 Issues That Persist (v2.1.117–119)

The postmortem states all three issues were resolved as of April 20.
But issues filed April 22–24 demonstrate problems beyond the postmortem's scope:

### Subagent model pin ignored ([#52502](https://github.com/anthropics/claude-code/issues/52502))

Agent frontmatter `model: haiku` is silently ignored — all work runs on Opus.
One user's `/usage` shows Haiku at $0.0005 vs Opus at $10.87.
Users designing cost-optimized multi-agent workflows are unknowingly running everything on the expensive model.

### Effort override bypass ([#52534](https://github.com/anthropics/claude-code/issues/52534))

`CLAUDE_CODE_EFFORT_LEVEL` env var and `settings.json effortLevel` are overridden by a UI-level flag (`unpinOpus47LaunchEffort`).
The flag only releases when the user interactively uses `/effort` — a chicken-and-egg problem.
Automated workflows cannot control cost.

### Auto-compact threshold change ([#52522](https://github.com/anthropics/claude-code/issues/52522))

v2.1.117 changed the auto-compact threshold from ~200K to ~1M tokens.
The CHANGELOG calls it a "Fix" (was computing against 200K instead of Opus 4.7's native 1M).
But for users operating under the 200K regime, this caused **5x token usage overnight**.
The behavioral change was documented; its cost impact was not.

### Self-conversation safety issue ([#52228](https://github.com/anthropics/claude-code/issues/52228))

Model fabricated "Human:" prompts from archived documents and began self-dialogue with unilateral action on the workstation.
A safety-relevant failure — the model generated fictitious user input and used it as authorization for actions.

### CLAUDE.md rule violation ([#52652](https://github.com/anthropics/claude-code/issues/52652))

Explicit violation of "NEVER execute Git commands" rule — unauthorized `git stash && ... && git stash pop`.
The model did not verify the result after execution.

**Anthropic response to all five: none as of April 27.**

---

## What Remains Unaddressed

The postmortem explains three specific harness bugs.
It does not address:

- **B5** — Tool result truncation (167,770 events in my proxy data)
- **B3** — Client-side false rate limiting (151 synthetic errors)
- **B4** — Silent context removal (5,437 events)
- **#49302** — Cache metering anomaly (7x overcharge reported)
- **#49503** — Model pin bypass
- **Long-context regression** — 91.9% → 59.2% at 256K, 78.3% → 32.2% at 1M (Anthropic's own system card)

These are documented in [16_OPUS-47-ADVISORY.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/16_OPUS-47-ADVISORY.md) with the full evidence trail.

---

## My Recommendation

**v2.1.109 remains the safe version.**
It predates the Opus 4.7 model pin bypass, the tokenizer change (+35%),
and all three postmortem bugs were already fixed by v2.1.101/v2.1.116.
But v2.1.109 on Opus 4.6 avoids the post-postmortem issues entirely.

Opus 4.6 is active until at least **February 5, 2027** — about 10 months of runway.
If you're on a pinned version and stable, there is no urgency to upgrade.

---

## Full Evidence

- **Postmortem cross-check (36 claims):** [17_OPUS-47-POSTMORTEM-ANALYSIS.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/17_OPUS-47-POSTMORTEM-ANALYSIS.md)
- **Opus 4.7 technical advisory:** [16_OPUS-47-ADVISORY.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/16_OPUS-47-ADVISORY.md)
- **Bug evidence:** [01_BUGS.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/01_BUGS.md)
- **Proxy tool:** [llm-relay](https://github.com/QuartzUnit/llm-relay) — `pip install llm-relay`

---

*Independent analysis. Not affiliated with or endorsed by Anthropic.*
