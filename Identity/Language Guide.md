**File Name**: language_guide **Phase**: All phases **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** defines the vocabulary and tone used throughout Spindle. Every string in the app — buttons, labels, notifications, messages — must follow this guide. Consistency builds the product identity.

---

## The Vocabulary Boundary — Read This First

Spindle has two distinct vocabularies that must never be mixed:

**Internal vocabulary** — used in code, design docs, service names, model fields, repository operations, and all developer-facing text. Precise, technical, unambiguous. This is the language of the implementation.

**UI vocabulary** — used in all user-facing strings: screen labels, buttons, notifications, messages, copy, onboarding, and marketing. This is the language of the weaving metaphor. Users never see the word "commitment" in isolation — they see "thread." Users never see "instance" — they see the garment or the day.

**The rule:** Claude generating code uses internal vocabulary for variable names, function names, and comments. Claude generating UI strings uses UI vocabulary. Never mix them in the same context.

---

## Internal Vocabulary

|Internal term|Meaning|
|---|---|
|Commitment|A habit the user is tracking — do or avoid|
|CommitmentDefinition|The template/blueprint for a commitment|
|CommitmentInstance|One occurrence of a commitment in a time window|
|LogEntry|One recorded action against an instance|
|livePerformance|Numeric completion percentage of an instance|
|completionPercent|The garment's visual completion, separate from livePerformance|
|isFrozen|Whether a commitment is paused|
|isCompleted|Whether a commitment is permanently finished|
|streak|Consecutive kept windows counter|
|WeeklyCup|The reward record for a strong week|
|ProgressionProfile|The user's level and point accumulation record|

---

## UI Vocabulary

|UI term|Maps to|Usage context|
|---|---|---|
|Thread|Commitment|Creation, list, general reference|
|Spin a thread|Create a commitment|Creation action|
|Garment|The visual on commitment detail screen|Progress representation|
|Weaving|Making progress on a do commitment|Progress context|
|Unraveling|Making progress on an avoid commitment|Avoid habit context|
|Thread woven|Commitment kept|Success moment|
|Thread skipped|Commitment missed|Miss moment — never "failed"|
|Kept streak|Streak count|Streak display|
|Weave score|Day or period score|Score display|
|How you're weaving|Analytics or performance|Section headers|
|Weave review|AI report|AI feature label|
|Understand this pattern|Deep AI analysis|AI feature trigger|
|Frozen|isFrozen: true|Status label|
|Complete|isCompleted: true|Status label|
|Rope|All commitments combined, overall progress|Dashboard, weekly summary|

---

## Core Equivalence

A **commitment** (internal) and a **thread** (UI) are the same thing.

- "Spin a new thread" — natural in creation context
- "Your threads" — natural in list context
- "This week's thread" — natural in weekly framing

Both internal and UI terms are correct in their respective contexts. They must not appear together in the same user-facing string.

---

## Garment Vocabulary

|Moment|Copy|
|---|---|
|Garment gaining threads (do habit, good day)|"Your weave grew stronger."|
|Garment losing threads (avoid habit, good day)|"Another thread unraveled."|
|Garment decay after 2 failed days|No copy — silence. The garment reflects the change visually.|
|Garment at 100% (do habit complete)|"The thread holds."|
|Garment at 0% (avoid habit grip minimal)|"The grip is gone. Keep the thread loose."|
|Garment highlighted new threads (Phase 3)|"This week's work."|

Decay is never announced. The garment reflects it silently — consistent with the UX principle that bad days receive silence, not punishment.

---

## Weaver Level Vocabulary

Level names and one-liners are the authoritative source. They are used verbatim in the app — never improvised or paraphrased. `EncouragementService._levelOneLiner()` must match this table exactly.

|Level|Name|One-liner|
|---|---|---|
|0|Apprentice|Every master weaver was an apprentice one day.|
|1|Weaver|The first threads are in place. The craft has begun.|
|2|Journeyman|The pattern is starting to show.|
|3|Artisan|Consistent craft, recognizable work.|
|4|Craftsman|Mastery is taking shape, thread by thread.|
|5|Master Weaver|Rare discipline. The loom obeys.|
|6|Loom Keeper|You tend the craft itself now.|
|7|Penelope|Twenty years of discipline — thread by thread.|

Level-up display copy (overlay and notification) is owned by `EncouragementService` — it reads the one-liner from a static map that mirrors this table. Any change to level copy must update both this doc and the static map in `EncouragementService._levelOneLiner()`.

---

## Tone Rules

**Analytical, never moral.** Every message describes what happened — it never evaluates the user as a person.

**Encouraging, never punishing.** Misses are neutral facts, not failures. Silence is better than a negative message.

**Specific, never generic.** "You skipped 3 threads this week" beats "You could do better."

**Craftsman-like, never cheerful.** The voice is calm and serious — the tone of someone who respects the work. Never excited, never celebratory in a hollow way.

---

## The Weaving Metaphor — When to Use It

Use thread/weaving language in copy that benefits from it. Do not force it into every string.

**Good contexts:**

- Onboarding ("start weaving")
- Streak milestones ("7 threads woven in a row")
- Weekly report identity statement ("your rope is getting stronger")
- End-of-day message ("thread woven")
- Level-up moments ("the craft has begun")
- Garment progress copy

**Neutral contexts — plain language works better:**

- Settings screens
- Error messages
- Technical confirmations
- Dates, times, numbers

---

## Do / Avoid Direction

- **Do** — something to build. Garment begins empty and grows complete. Use "weave" and "build" in copy.
- **Avoid** — something to unravel. Garment begins complete and grows empty. Use "unravel" and "loosen" in copy.

In the UI these appear as section headers: **DO** and **AVOID** — plain, clear, no metaphor needed at the structural level.