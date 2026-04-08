**File Name**: later_improvements **Phase**: Post-Phase 3 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** ideas agreed upon but deferred beyond Phase 3. Each item is self-contained and can be added without affecting existing features.

---

## List

**Rewards feature.** A `RewardService` that evaluates sustained rolling performance over a rolling period (e.g. 30 days above threshold) and calls `AchievementService.addAchievement()` with `type: reward, subtype: periodicReward`. Sits above Achievements in the chain. Zero changes to any existing feature when added — `AchievementSubtype.periodicReward` and `AppConfig.pointsPeriodicReward` are already present as placeholders. See `service_achievement.md` Later Improvements.

**Garment fortify phase.** After a garment completes, it enters a fortify phase — the user keeps building and the garment gains a visual reinforcement layer (golden thread border). Completing fortify calls `AchievementService.addAchievement()` with a `garmentFortified` subtype. Requires a `GarmentPhase` enum field on `GarmentProfile` and a new `AchievementSubtype` value. See `service_garment.md` Later Improvements.

**Garment gold phase.** After fortify, a gold phase begins — full gold visual treatment representing mastery. Completing it records `garmentGolded`. Each phase produces its own achievement and point value. Makes long-running commitments feel progressively rewarding rather than plateauing at completion. See `service_garment.md` Later Improvements.

**Success threshold per commitment.** Minimum % to count as kept. Currently defaults to 0% (any progress counts). Adding a configurable threshold per commitment would allow "only counts if I reach 80% of my walk target." Field already exists on `CommitmentDefinition` — UI is the only addition needed.

**Commitment templates.** Pre-built common commitments for both do and avoid types. Speeds up onboarding and helps users who don't know where to start.

**Export data.** CSV or JSON export of full history — all instances, log entries, and weekly cups. Useful for power users and builds trust ("your data is yours").

**iOS support.** After Android is stable and validated. Same Flutter codebase, minimal changes needed.

**Global freeze.** Freeze all active commitments at once from Settings. Single action for travel or illness without touching each one individually.

**Multiple scheduled freeze periods.** Pre-schedule several freeze periods in advance — e.g. freeze Nov 10–17 and Dec 24–Jan 1 simultaneously. Useful for predictable annual patterns (Ramadan, holidays).

**Notary Mode.** Generate a read-only share link to Your Record for a trusted person (spouse, coach, accountability partner). Requires Phase 3 Firebase. The notary sees Today Score and identity statement only — cannot edit anything.

**Dependency Map.** Visual graph showing correlations between commitments — e.g. "sleep before 11pm" connected by a thick line to "morning walk" when correlation is strong. Teaches behavioral architecture: users discover root causes, not just symptoms. Requires Phase 2 AI + several weeks of data.

**Pay-per-insight.** For free users who exceed the micro-insight limit, offer a one-time purchase for additional insights instead of forcing an upgrade. Reduces friction for casual users.

**Adaptive difficulty.** Suggest raising or lowering commitment targets based on recent performance. See `component_adaptive_difficulty.md`.

**Home screen widget.** Android home screen widget with Today Score and quick-log buttons. See `component_home_screen_widget.md`.

**Commitment description field.** Already added to `CommitmentDefinition` and Stage 3 of the form. UI polish only.

**Sharer reward.** Give the sharer subscription credit when a referred user activates. The infrastructure (referral code, activation tracking via Cloud Function) is already built in Phase 3. Requires a new row in the referral summary and a RevenueCat credit call — no UI redesign.