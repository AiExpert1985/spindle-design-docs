**File Name**: readme
**Phase**: All phases
**Created**: 15-Mar-2026
**Modified**: 21-Mar-2026

---

# Spindle

Commitment is the thread. Consistency is the weaving. Character is the rope.

Spindle is a personal accountability app built around the metaphor of weaving. Each day you keep a commitment, you add a thread to your garment. Each day you resist a bad habit, one thread unravels from it. The garment is always honest — it reflects real consistency, not just accumulated count.

**Platform:** Android first (Flutter) · **Tagline:** Weave your life, one thread at a time.

---

## The Problem Spindle Solves

You know the target. You have the plan. You know the how. And yet nothing changes.

The missing piece is never the goal or the plan — it is commitment over time. Quiet, daily, unglamorous consistency. Spindle solves this through honest tracking, the garment metaphor, and a progression system that recognizes real craft.

---

## Tech Stack

| Layer | Choice |
|---|---|
| Platform | Flutter — Android first |
| State management | Riverpod |
| Navigation | GoRouter |
| Local DB | Drift (SQLite) — free tier |
| Cloud DB | Firebase Firestore — Pro/Premium |
| Auth | Firebase Auth — Phase 3 |
| Payments | RevenueCat — Phase 3 |
| Notifications | flutter_local_notifications + WorkManager |
| AI | Anthropic API — Phase 3, paid tier only |
| Event bus | Dart broadcast streams |

---

## Build Phases

**Phase 1 — MVP (local only)**
Core commitment tracking, activity logging, rewards, notifications. Zero cloud dependency. For internal testing.

**Phase 2 — Intelligence and Engagement**
Performance accounting, garment visuals, analytics, onboarding, settings, share progress. Real users.

**Phase 3 — Cloud, AI and Monetization**
Firebase sync, AI insights, weekly summaries, subscriptions, auth, progression system, referrals.

**Phase 4 — Community**
Story sharing, community channel, public profiles, follow system. See `feature___community` for the full plan.

---

## Architecture in One Paragraph

Spindle is organized into features. Each feature has one responsibility, owns its own model, service, and repository, and communicates with other features through two mechanisms only: service calls (for reads) and events (for reactions). Dependencies flow in one direction — a feature may only depend on features below it in the chain. The presentation layer is passive — it observes Riverpod providers and forwards user intent to services. It never calculates or decides.

---

## Key Reference Docs

Start here when making any design or implementation decision.

| Doc | What it answers |
|---|---|
| `architecture_rules` | All rules every feature must follow |
| `feature_dependency_chain` | Which features depend on which, what each exposes, violation examples |
| `event_catalog` | All events in the system and what they carry |
| `documentation_rules` | How to write and maintain project docs |
| `language_guide` | Internal vs UI vocabulary, tone, weaving metaphor |
| `database_architecture` | Drift vs Firestore, migration, storage rules |

---

## Language

Two vocabularies — one for code, one for users. Never mix them.

- **Internal** (code, docs): commitment, instance, logEntry, livePerformance
- **UI** (user-facing): thread, weaving, garment, unraveling, weaver

See `language_guide` for full vocabulary and tone rules.
