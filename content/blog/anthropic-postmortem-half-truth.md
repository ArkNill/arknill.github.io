---
title: "Anthropic's Postmortem Told Half the Truth"
date: 2026-04-27
description: "Anthropic admitted 3 harness bugs caused Claude Code degradation. The bugs are real. But the postmortem strategically scoped out model-level regressions, ignored 9+ open issues, and all three 'bugs' happen to reduce Anthropic's serving costs. Here's what the other half looks like."
tags: ["claude-code", "ai", "anthropic", "postmortem", "transparency"]
showToc: true
TocOpen: false
draft: false
---

On April 23, Anthropic published [a postmortem](https://www.anthropic.com/engineering/april-23-postmortem) acknowledging three product-layer bugs that degraded Claude Code from March 4 through April 20.
They frame it as: model weights unchanged, harness bugs fixed, problem solved.

**The three bugs are real. Their impact was real. The fixes were real.**

But the postmortem is a carefully scoped document that tells half the truth. Here's the other half.

---

## What They Admitted (Correctly)

| Bug | Introduced | Fixed | Duration |
|-----|-----------|-------|----------|
| Effort `high` → `medium` | March 4 (v2.1.68) | April 21 (v2.1.117) | **48 days** |
| Thinking cache clear every turn | March 26 (v2.1.85) | April 10 (v2.1.101) | 15 days |
| "≤25 words" system prompt | April 16 (v2.1.111) | April 20 (v2.1.116) | 4 days |

All three are product-layer issues — API parameters and system prompts, not model weights.
The combined effect: effort reduced thinking depth, thinking cache destroyed session memory, word limit truncated output.
Together, these made Claude appear significantly dumber.

This explanation is technically correct.
A user hitting all three bugs simultaneously would experience exactly the degradation that was reported.

I accept this part.

---

## The Strategic Scoping

The postmortem covers "why 4.6 users felt degradation." It does not address a separate, concurrent complaint stream: **"4.7 is worse than 4.6."**

These are model-level regressions that have nothing to do with harness bugs:

| Opus 4.7 Model Issue | Evidence | Harness Bug? |
|---------------------|----------|:---:|
| Tokenizer +35% (worse on CJK) | [Official docs](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7): "1.0x to 1.35x more tokens" | No �� architecture change |
| Long-context retrieval: 91.9% → 59.2% (256K) | Anthropic's own system card (MRCR v2) | No — model capability |
| Long-context retrieval: 78.3% → 32.2% (1M) | Anthropic's own system card (MRCR v2) | No — model capability |
| Literal instruction following | [What's New](https://platform.claude.com/docs/en/about-claude/models/whats-new-claude-4-7): "follows instructions more literally" | No — training change |
| BrowseComp: 84.0% → 79.6% | Anthropic's benchmark | No — model regression |
| XML/JSON mixing in long payloads ([#49747](https://github.com/anthropics/claude-code/issues/49747)) | Decoder-level format switching | No — model behavior |
| Safety over-refusal ([#49751](https://github.com/anthropics/claude-code/issues/49751)) | Structural biology blocked on 4.7, works on 4.6 | No — RLHF/safety layer |

The postmortem mentions none of these.
Zero lines.
The narrative is: "we fixed three bugs, Claude is back to normal."
But users who upgraded to 4.7 have a different set of problems that three bug fixes don't address.

**The postmortem solves Stream A ("4.6 got dumber") while pretending Stream B ("4.7 is a regression") doesn't exist.**

---

## The Cost Coincidence

All three admitted bugs share a common direction:

| Bug | User Impact | Anthropic Serving Cost |
|-----|------------|----------------------|
| Effort `high` → `medium` | Quality reduction | **Compute ↓↓** |
| Thinking cache clear every turn | Context loss, repetition | **Prefill ↓↓** (less to cache) |
| "≤25 words" output limit | Information reduction | **Output tokens ↓↓** |

Three changes, all independently reducing Anthropic's cost of serving each request.
Framed as "UI freezing fix," "caching optimization," and "verbosity reduction."

I cannot prove intent. But:
- The probability of 3/3 changes accidentally aligning with cost reduction is low
- Two of three were **never recorded in the CHANGELOG** — suggesting organizational awareness that they'd be controversial
- The effort change was recorded but framed as a product improvement ("sweet spot between speed and thoroughness"), never as a cost-driven trade-off

If these were genuine accidents with no cost motivation, why hide two of them from the changelog entirely?

---

## The CHANGELOG Gap Is the Real Story

Forget the bugs for a moment. The structural finding is:

> **Anthropic can and does make behavior-affecting changes to Claude Code without any public record.**

The thinking cache bug was introduced March 26 and fixed April 10.
Neither event appears in the CHANGELOG.
I searched all 3,285 lines — the terms "thinking cache," "cache clear," and "thinking prun" appear **nowhere**.

The "≤25 words" system prompt was added April 16 and reverted April 20.
Neither event appears in the CHANGELOG.
The terms "25 words" and "length limit" appear **nowhere**.

System prompt changes are structurally invisible. Users cannot:
- See what system prompts are active in their version
- Diff system prompts between versions
- Know when a system prompt change degraded their experience
- Distinguish "the model is dumber" from "an invisible instruction is limiting it"

The postmortem's remediation plan mentions "soak periods" and "gradual rollouts" — but says nothing about **mandatory documentation** for behavior-affecting changes.
The structural opacity remains intact.

---

## Post-Postmortem: Not Fixed

The postmortem narrative is "three bugs, all fixed as of April 20." Issues filed April 22–24 on v2.1.117–119 show otherwise:

| Issue | Problem | Anthropic Response |
|-------|---------|-------------------|
| [#52502](https://github.com/anthropics/claude-code/issues/52502) | Subagent `model: haiku` pin ignored — everything runs on Opus ($10.87 vs $0.0005) | None |
| [#52534](https://github.com/anthropics/claude-code/issues/52534) | `effortLevel` setting overridden by `unpinOpus47LaunchEffort` flag — programmatic control broken | None |
| [#52522](https://github.com/anthropics/claude-code/issues/52522) | Auto-compact threshold changed from 200K→1M — 5x token usage overnight | Bot: "duplicate" |
| [#52228](https://github.com/anthropics/claude-code/issues/52228) | Model fabricated "Human:" prompts, began self-dialogue with unilateral workstation action | None |

These are not edge cases.
#52502 is a billing issue with direct financial impact.
#52534 means effort configuration doesn't work.
#52522 means a "fix" caused 5x cost increase for existing users.

The "all fixed" framing lasted approximately 48 hours before new issues contradicted it.

---

## What I'd Respect Instead

If the postmortem had said:

> *"We found three harness bugs that explain the 4.6 degradation reports. We've fixed them. Separately, Opus 4.7 has known trade-offs (tokenizer efficiency, long-context retrieval, instruction following behavior) that we're working on. We also acknowledge that our CHANGELOG does not cover system prompt changes, and we're evaluating how to improve this."*

That would be a complete postmortem.
What they published is **incident response for the defensible subset** — an engineering blog post doing the job of a PR statement.

---

## My Position

I run Claude Code on a pinned version (v2.1.109) with Opus 4.6 and have since April 15.
The three postmortem bugs are:
- Bug #1 (effort): affected v2.1.68+, so my version was exposed March 4–April 21 (**48 days**)
- Bug #2 (thinking cache): affected v2.1.85–101, my version was exposed March 26–April 10 (**15 days**) before it was fixed server-side in v2.1.101
- Bug #3 (≤25 words): v2.1.111+, never hit my pinned version

One of three bugs affected me.
The other two didn't because I was already pinned below them.
This validates version pinning as a defensive strategy — and the postmortem itself confirms that **Anthropic ships behavior-affecting changes without notice**,
which is exactly why pinning exists.

Opus 4.6 remains active until at least [February 5, 2027](https://platform.claude.com/docs/en/about-claude/model-deprecations).
I have 10 months of runway.
The postmortem didn't change my decision — it reinforced it.

---

## The Real Fix: Official LTS for AI Developer Tools

I shouldn't have to do this.
Version pinning, proxy monitoring, manual CHANGELOG diffing — these are user-side defenses against a problem that vendors should solve structurally.

**Every company shipping AI-powered CLI/IDE tools needs an official LTS (Long-Term Support) channel.**
Not just Anthropic — this applies to Cursor, GitHub Copilot, Windsurf, Cline, and every tool that intercepts a developer's workflow.

### Why User-Side Pinning Is Not Enough

My setup works: pin v2.1.109, set `DISABLE_AUTOUPDATER=1`, monitor with a proxy.
But this approach:

- **Doesn't scale.** Most users don't run transparent proxies or read CHANGELOGs. They just notice "it got worse" and can't diagnose why.
- **Is adversarial.** I'm fighting my own tool's update mechanism. The tool actively wants to be on latest — auto-updater, deprecation timers, feature flags that assume current version.
- **Has an expiration date.** Opus 4.6 retires February 2027. Old client versions will eventually hit API incompatibilities. There's no guarantee a pinned version keeps working.
- **Shifts responsibility to users.** If a vendor ships a breaking change, the user who didn't pin is blamed for not pinning. That's backwards.

### What Official LTS Looks Like

The model already exists in every serious software ecosystem — Node.js, Ubuntu, PostgreSQL, Java.
For AI developer tools:

| Feature | Current Reality | LTS Channel |
|---------|----------------|-------------|
| Update cadence | Continuous (daily releases) | Quarterly security/stability patches only |
| Model changes | Any time, no opt-in | Frozen model for LTS duration, explicit migration path |
| System prompt changes | Invisible, unrecorded | Documented, opt-in for LTS users |
| Breaking behavior changes | Shipped without notice | Require explicit migration (deprecation → removal cycle) |
| Minimum support window | "Not sooner than" deprecation date | **Guaranteed** 12-month minimum from LTS release |
| Rollback | User responsibility (pin + proxy) | Vendor-provided: `claude --channel lts` or equivalent |

### The Business Case

This isn't altruism — it's retention:

- **Enterprise trust.** No enterprise will depend on a tool that silently changes behavior. Official LTS = enterprise-ready. Without it, every procurement team adds "vendor instability" to their risk matrix.
- **Reduced support load.** Half of Anthropic's 100+ Opus 4.7 issues are users confused by unannounced changes. A stable channel eliminates this category entirely.
- **Competitive differentiation.** The first AI coding tool to offer genuine LTS wins the "production infrastructure" market segment. Right now, all of them are optimized for demo-day, not day-200.

### What I'm Asking For

To Anthropic, and to every company shipping AI developer tools:

1. **An official LTS channel** with a guaranteed support window (12+ months)
2. **No invisible behavior changes** on the LTS channel — every change documented, no stealth system prompt modifications
3. **Model version guarantee** — LTS users stay on the model they chose until they explicitly migrate
4. **Rollback mechanism** — not "pin a version and hope," but vendor-supported channel switching
5. **Transparent CHANGELOG** that covers system prompt changes, not just client code changes

The current situation — where users must reverse-engineer proxy data to understand why their tool degraded —
is not a sustainable relationship between vendors and professional developers.

---

## Evidence

- **Full cross-check (36 claims verified):** [17_OPUS-47-POSTMORTEM-ANALYSIS.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/17_OPUS-47-POSTMORTEM-ANALYSIS.md)
- **Opus 4.7 technical advisory:** [16_OPUS-47-ADVISORY.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/16_OPUS-47-ADVISORY.md)
- **42K API call analysis:** [01_BUGS.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/01_BUGS.md)
- **DATASET.md** (4 independent datasets): [DATASET.md](https://github.com/ArkNill/claude-code-hidden-problem-analysis/blob/main/DATASET.md)

---

*Independent analysis based on 42,363 proxy-captured API calls, 36-claim cross-check of the postmortem, and 9 open GitHub issues. Not affiliated with or endorsed by Anthropic.*
