**File Name**: service_encouragement **Feature**: Encouragement **Phase**: 2 **Created**: 18-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** produces all ephemeral encouragement responses. Purely reactive — subscribes to events and emits display signals via Riverpod providers. No persistent storage. Nothing else in the app produces encouragement responses.

---

## Design

Encouragement is a read-only reactor. It subscribes to events from below, reads what it needs to select the right response, and emits a signal. The signal is ephemeral — consumed once by the presentation layer and cleared. Nothing is stored except `lastEncouragementType` on `UserSettingsProfile` for story deduplication, written through `UserSettingsService`.

---

## Events Subscribed

### `Heartbeat.longIntervalTick` → `_onTick(tick)`

Day celebration Path B — notification at configured time.

```
if not _temporal.isNearTime(tick.timestamp, userPrefs.celebrationTime): return
if not TickGuard.shouldRun('celebration', today): return
if not userPrefs.celebrationEnabled: return

TickGuard.markRan('celebration', today)
_evaluateDayCelebration(today)
```

Fires at most once per day via `TickGuard`.

### `ActivityEvent` where `type: created` → `_onActivityRecorded(event)`

Immediate feedback after a positive log. Used by the Dashboard card — deferred to Phase 2 when the Dashboard interaction is designed.

```
if livePerformance did not increase: return   // no feedback on miss

emit LogFeedbackSignal(livePerformance)

// threshold crossings — checked in memory, fire once per session per threshold
if crossed 50%:  emit ThresholdMessageSignal("Halfway there.")
if crossed 100%: emit ThresholdMessageSignal("Commitment kept.")
```

### `InstanceUpdatedEvent` where `snapshot.status == closed` → `_onWindowClosed(event)`

Path A — in-app day celebration. Checks if all commitment windows for today are now closed.

```
allClosed = CommitmentIdentityService.getInstancesForDay(today)
              .all(instance => instance.status == closed)

if not allClosed: return

dayScore = PerformanceService.getDayScore(today)
if dayScore < userPrefs.celebrationThreshold: return
if TickGuard.hasRun('celebration', today): return   // already celebrated today

TickGuard.markRan('celebration', today)
_evaluateDayCelebration(today)
```

### `AchievementEarnedEvent` where `record.type == streakMilestone` → `_onMilestone(event)`

```
emit MilestoneOverlaySignal(
  streakCount: int.parse(event.record.subtype.split('_')[0]),
  definitionId: event.record.definitionId
)
```

### `AchievementEarnedEvent` where `record.type == cup` → `_onCupEarned(event)`

```
emit CupCelebrationSignal(
  cupLevel: event.record.subtype,
  earnedAt: event.record.earnedAt
)
```

Shown when user opens app after Sunday calculation.

### `LevelReachedEvent` → `_onLevelReached(event)`

```
emit LevelUpOverlaySignal(
  newLevel: event.newLevel,
  levelName: event.levelName,
  oneLiner: event.levelUpMessage
)
```

Message copy comes from the event — `ProgressionService` puts it there so Encouragement never calls back.

---

## Emitted Signals (Riverpod providers)

All signals are ephemeral — consumed once by the presentation layer and cleared.

```
LogFeedbackSignal
  livePerformance: double
  definitionId: String

ThresholdMessageSignal
  message: String

MilestoneOverlaySignal
  streakCount: int
  definitionId: String

CupCelebrationSignal
  cupLevel: String
  earnedAt: DateTime

LevelUpOverlaySignal
  newLevel: int
  levelName: String
  oneLiner: String

DayCelebrationSignal
  dayScore: double
  storyType: int
  storyText: String
```

---

## Day Celebration Story Selection

### `_evaluateDayCelebration(date)`

```
dayScore = PerformanceService.getDayScore(date)
facts = AnalyticsService.computeDayFacts(date)
prefs = UserCoreService.getProfile()
lastType = UserSettingsService.getLastEncouragementType()

storyType = _selectStory(facts, lastType)
storyText = _buildStoryText(storyType, facts, dayScore)

UserSettingsService.recordEncouragementSent(storyType)
emit DayCelebrationSignal(dayScore, storyType, storyText)
```

Seven story types selected by priority. Type 6 always applies as fallback. Never repeats the same type two consecutive days — deduplication via `lastEncouragementType` on `UserSettingsProfile`.

|Priority|Type|Example|
|---|---|---|
|1|Comeback|"After a tough few days, you're back at 71%."|
|2|Improvement over yesterday|"Up from 61% to 74%. Moving forward."|
|3|Commitment spotlight|"You walked every day this week."|
|4|Consistency (3+ weeks)|"Three weeks of quiet discipline."|
|5|Partial progress|"3km of your 5km walk. Partial progress moves you forward."|
|6|Overall score (fallback)|"83% today. Your garment grew stronger."|
|7|Slight setback (still ≥ threshold)|"Slightly below yesterday, but 65% still counts."|

---

## Rules

- No persistent storage — `lastEncouragementType` on `UserSettingsProfile` is the only exception, written through `UserSettingsService`
- Tick subscription uses `TickGuard` — celebration fires at most once per day
- `LogFeedbackSignal` and `ThresholdMessageSignal` deferred to Phase 2 (Dashboard card interaction not yet designed)
- Threshold crossings tracked in memory — fire once per crossing per session, never stored
- Signals are ephemeral — never replayed

---

## Dependencies

- `Heartbeat` — subscribes to `longIntervalTick`
- `CommitmentService` — subscribes to `InstanceUpdatedEvent`
- `ActivityService` — subscribes to `ActivityEvent`
- `AchievementsService` — subscribes to `AchievementEarnedEvent`
- `ProgressionService` — subscribes to `LevelReachedEvent`
- `TickGuard` — day celebration idempotency
- `TemporalHelperService` — near-time check for celebration time
- `PerformanceService.getDayScore()` — day score for celebration threshold check
- `CommitmentIdentityService.getInstancesForDay()` — checks all windows are closed (Path A)
- `AnalyticsService.computeDayFacts()` — story selection
- `UserCoreService.getProfile()` — celebration preferences
- `UserSettingsService.recordEncouragementSent()` — story deduplication