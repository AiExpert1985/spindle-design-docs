**File Name**: feature_ai_insights **Phase**: 3 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

**Availability:** Pro / Premium only.

---

AI Insights takes the structured facts that Analytics computes and turns them into genuine AI reasoning — causes, patterns, and specific actionable recommendations. It is the feature that answers the questions a performance score cannot: why is this commitment hard on Fridays? What is the one thing most likely to move the needle?

---

## Why It Exists

Numbers describe what happened. AI reasoning explains it. A user who sees "64% this week" knows something is wrong but not why. An insight that says "you tend to skip your evening walk on days with back-to-back afternoon logs — the pattern shows up on 7 of the last 9 misses" is actionable. That is the gap this feature fills.

The AI receives no user identifiers — only behavioral facts and anonymized notes. It is analytical, never moral. It describes patterns, never judges the person.

---

## Position in the System

Sits above Analytics, which it calls for pre-computed facts. Also depends on UserSettings for quota management. Pro/Premium only — always on Firestore, no Drift implementation needed.

---

## How It Works

Three insight types, each serving a different need:

**Micro-insight** — one commitment, last 30 days. The most personal insight. Shows causes, failure patterns, and one or two specific recommendations. Available to all tiers within a weekly quota (free users have a configurable limit; Pro/Premium have none).

**Quick summary** — all commitments, one week. A cross-commitment narrative of the week. No notes included — facts only. Pro/Premium only.

**Deep report** — all commitments, custom period. Full analysis with notes and cross-commitment patterns. Rate-limited to once per 7 days. Pro/Premium only.

**Why Analytics computes first, AI receives facts.** The AI never touches raw instances or log entries. `AnalyticsService` produces a clean, structured payload — stats, patterns, notes — and that is what goes into the prompt. This keeps prompts consistent, testable, and independent of how data is stored. Swapping the AI provider or changing the prompt requires no data access changes.

**Storage policy.** One record per insight type per commitment (micro-insight), one record total for quick summary and deep report. New generation always overwrites — no history is kept. The user is told explicitly that the previous insight will be replaced. Maximum records at any time: number of active commitments + 2.

**Prompt tone.** All prompts are analytical, never moral. Specific, never generic. One or two recommendations maximum — more overwhelms and none get acted on.

**Provider abstraction.** `AIInsightService` is an abstract interface. The current implementation uses Anthropic's API. Swapping to another provider means adding one new implementation class — nothing else changes.

---

## Rules

- Pro/Premium only — no Drift implementation
- Input is always pre-computed facts from Analytics — never raw data
- AI provider receives no user identifiers — behavioral facts and anonymized notes only
- Rate limits and quota enforcement happen here — components never enforce limits themselves
- Components never construct prompts — the service formats the payload internally
- All functions return `Result<T>` — quota exhausted, rate limited, and API errors all surface to the presentation layer
- New generation always overwrites — no insight history maintained
- `type + definitionId` is the unique storage key

---

## Later Improvements

**Additional providers.** OpenAI, Gemini, or others. One new implementation class each — zero changes to the interface or calling code.

**Insight history.** Optional — keep previous insights rather than overwriting. Requires a storage policy change and user-facing UI to browse history.