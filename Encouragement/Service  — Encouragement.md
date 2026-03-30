**File Name**: service_encouragement **Feature**: Encouragement **Phase**: 2 **Created**: 18-Mar-2026 **Modified**: 28-Mar-2026

---

**Purpose:** produces all ephemeral encouragement signals. Purely reactive — subscribes to events and emits display signals via Riverpod providers. No persistent storage except `lastEncouragementType` on `UserSettingsProfile`, written through `UserSettingsService`.

---

## Events Subscribed

### `Heartbeat.longIntervalTick` → `_onTick(tick)`

Day celebration Path B — fires at configured celebration time.

```
if not isNearTime(tick.timestamp, userPrefs.celebrationTime): return
if dayCelebrationFiredToday: return   // in-memory guard
if not userPrefs.celebrationEnabled: return

dayCelebrationFiredToday = true
_evaluateDayCelebration(today)
```

### `ActivityEvent` → `_onActivityRecorded(event)`

Immediate feedback after a positive log. Deferred to Phase 2.

```
if livePerformance did not increase: return

emit LogFeedbackSignal(livePerformance, event.definitionId)

// threshold crossings — tracked in memory, fire once per session per threshold
if crossed 50%:  emit ThresholdMessageSignal("Halfway there.")
if crossed 100%: emit ThresholdMessageSignal("Commitment kept.")
```

### `InstanceUpdatedEvent` where `snapshot.status == closed` → `_onWindowClosed(event)`

Day celebration Path A — in-app trigger.

```
instances = CommitmentIdentityService.getInstancesForDay(today)
allClosed = instances.all(i => i.status == closed)
if not allClosed: return

dayScore = PerformanceService.getDayScore(today)
if dayScore < userPrefs.celebrationThreshold: return
if dayCelebrationFiredToday: return   // in-memory guard

dayCelebrationFiredToday = true
_evaluateDayCelebration(today)
```

---

## Internal Functions

### `_evaluateDayCelebration(date)`

```
dayScore = PerformanceService.getDayScore(date)
facts = _buildDayFacts(date)
lastType = UserSettingsService.getLastEncouragementType()

storyType = _selectStory(facts, lastType)
storyText = _buildStoryText(storyType, facts, dayScore)

UserSettingsService.recordEncouragementSent(storyType)
emit DayCelebrationSignal(dayScore, storyType, storyText)
```

### `_buildDayFacts(date)` → DayFacts

Reads from Performance and Activity to build the behavioral context needed for story selection — yesterday's score, recent streak, partial progress data. No external analytics service needed.

### `_selectStory(facts, lastType)` → int

Selects story type 1–7 by priority. Never returns `lastType` unless it is the only applicable option. Type 6 always applies as fallback.

### `_buildStoryText(storyType, facts, dayScore)` → String

Constructs the story string for the selected type using facts and score.

---

## Emitted Signals (Riverpod providers)

All ephemeral — consumed once and cleared.

```
LogFeedbackSignal
  livePerformance: double
  definitionId: String

ThresholdMessageSignal
  message: String

DayCelebrationSignal
  dayScore: double
  storyType: int
  storyText: String
```

---

## Rules

- No persistent storage — `lastEncouragementType` is the only written state
- Day celebration fires at most once per day — `dayCelebrationFiredToday` in-memory flag
- Threshold crossings tracked in memory — fire once per crossing per session
- Signals are ephemeral — never replayed
- Display copy lives here — not in any event or producing feature
- `LogFeedbackSignal` and `ThresholdMessageSignal` deferred to Phase 2

---

## Dependencies

- `Heartbeat` — subscribes to `longIntervalTick`
- `ActivityService` — subscribes to `ActivityEvent`
- `CommitmentIdentityService` — subscribes to `InstanceUpdatedEvent`; calls `getInstancesForDay()`
- `PerformanceService.getDayScore()` — day score
- `PerformanceService.getPerformanceForPeriod()` — recent performance for story context
- `UserCoreService.getProfile()` — celebration preferences
- `UserSettingsService.recordEncouragementSent()` — story deduplication write
- `UserSettingsService.getLastEncouragementType()` — story deduplication read