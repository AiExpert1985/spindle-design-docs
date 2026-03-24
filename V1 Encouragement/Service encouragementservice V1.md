**Created**: 18-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Encouragement **Phase**: 1

**Purpose:** produces all ephemeral encouragement responses. Purely reactive — subscribes to events and tick, emits display signals via Riverpod. No persistent storage. Nothing else produces encouragement responses.

---

## Events Subscribed

### `SchedulerTickEvent`

→ Day celebration Path B timing check.

```dart
eventBus.on<SchedulerTickEvent>().listen((tick) {
  if (!tick.isWakingHours) return;
  if (!_guard.shouldRun('celebration', tick.date)) return;

  final prefs = userService.getPreferences();
  if (!prefs.celebrationEnabled) return;
  if (!_isNearTime(tick.timestamp, prefs.celebrationTime)) return;

  _guard.markRan('celebration', tick.date);
  evaluateDayCelebration(tick.date);
});
```

Uses `TickGuard` with `tick.date` as period — fires at most once per day.

### `ActivityRecordedEvent`

→ `evaluateImmediateFeedback(event)`

Immediate log feedback after every user activity:

- `logResult: kept` and progress increased → emit `LogFeedbackSignal(type: positive)`
- Threshold crossings (50%, 100%, today score 80%) → emit `ThresholdMessageSignal(message)`
- `logResult: missed` → no signal — silence

### `PerformanceEntryWrittenEvent`

→ `evaluateDayCelebrationPathA(event)`

Path A — in-app: checks if all commitment windows for today are now closed. If yes and day score meets threshold → emit `DayCelebrationSignal`.

### `StreakMilestoneReachedEvent`

→ emit `MilestoneOverlaySignal(streakCount, badge, commitmentName)`

### `CupEarnedEvent`

→ emit `CupCelebrationSignal(cupLevel, weekStart)`

Shown when user opens app after Sunday calculation.

### `LevelReachedEvent`

→ emit `LevelUpOverlaySignal(newLevel, levelName, event.levelUpMessage)`

Message copy comes from the event payload — `ProgressionService` puts it there so Encouragement never calls ProgressionService.

---

## Emitted Signals (Riverpod providers)

```
LogFeedbackSignal         type, value, progressPercent
ThresholdMessageSignal    message
MilestoneOverlaySignal    streakCount, badge, commitmentName
CupCelebrationSignal      cupLevel, weekStart
LevelUpOverlaySignal      newLevel, levelName, oneLiner
DayCelebrationSignal      dayScore, storyType, storyText
```

Signals are ephemeral — consumed once by the presentation layer and cleared.

---

## Day Celebration Story Selection

Uses `AnalyticsService.computeDayFacts(today)` for story selection. Seven story types by priority — see `day_celebration.md` for the full table.

`UserService.recordEncouragementSent(type)` called after emitting `DayCelebrationSignal` to prevent the same story type repeating two consecutive days.

---

## No Repository

No persistent storage. One exception: `lastEncouragementType` on `UserProfile` — written by `UserService`, read by Encouragement through `UserService`.

---

## Rules

- Never calls feature services directly in response to events — reads only what is needed to emit the signal
- Tick subscription uses `TickGuard` — celebration fires at most once per day
- Condition checks on tick are instant — no async work before guard passes
- Signals are ephemeral — never stored, never replayed
- One threshold message per crossing per session — tracked in memory

---

## Dependencies

- `EventBus` — subscribes to events and tick, emits nothing via event bus (signals go to Riverpod)
- `TickGuard` — idempotency for tick-based day celebration
- `AnalyticsService` — day facts for celebration story
- `UserService` — celebration preferences, story deduplication
- Riverpod providers — destination for all emitted signals