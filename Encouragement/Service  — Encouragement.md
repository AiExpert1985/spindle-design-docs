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
if not isNearTime(tick.timestamp, userPrefs.celebrationTime): return
if not TickGuard.shouldRun('celebration', today): return
if not userPrefs.celebrationEnabled: return

TickGuard.markRan('celebration', today)
_evaluateDayCelebration(today)
```

Fires at most once per day via `TickGuard`.

### `ActivityService.onActivityCreated` → `_onActivityRecorded(event)`

Immediate feedback after a positive log. Used by the Dashboard card — deferred to Phase 2 when the Dashboard interaction is designed.

```
if livePerformance did not increase: return   // no feedback on miss

emit LogFeedbackSignal(livePerformance, event.definitionId)

// threshold crossings — tracked in memory, fire once per session per threshold
if crossed 50%:  emit ThresholdMessageSignal("Halfway there.")
if crossed 100%: emit ThresholdMessageSignal("Commitment kept.")
```

### `CommitmentService.onInstanceUpdated` where `snapshot.status == closed` → `_onWindowClosed(event)`

Path A — in-app day celebration. Checks if all commitment windows for today are now closed.

```
instances = CommitmentIdentityService.getInstancesForDay(today)
allClosed = instances.all(i => i.status == closed)

if not allClosed: return

dayScore = PerformanceService.getDayScore(today)
if dayScore < userPrefs.celebrationThreshold: return
if TickGuard.hasRun('celebration', today): return

TickGuard.markRan('celebration', today)
_evaluateDayCelebration(today)
```

**Why `InstanceUpdatedEvent where status: closed` instead of `PerformanceUpdatedEvent`:** `PerformanceUpdatedEvent` previously carried an `isClosed` field that no longer exists. Window close is a structural lifecycle event on the instance — `InstanceUpdatedEvent where status: closed` is the correct signal. Performance is then read via `PerformanceService.getDayScore()`.

### `AchievementService.onAchievementEarned` where `type: streak AND _isMilestoneSubtype(subtype)` → `_onMilestoneEarned(event)`

```
streakCount = _milestoneStreakCount(event.record.subtype)
emit MilestoneOverlaySignal(
  streakCount: streakCount,
  definitionId: event.record.definitionId,
)
```

Filters to `AchievementType.streak` with a milestone subtype. Uses a static map to convert `AchievementSubtype` enum to the streak count integer — no string parsing.

**Why no string parsing:** `AchievementSubtype` is a compile-time enum. The previous `'7_day'.split('_')[0]` approach was fragile string manipulation that breaks silently on rename. A static map is safe and explicit.

### `AchievementService.onAchievementEarned` where `type: cup` → `_onCupEarned(event)`

```
emit CupCelebrationSignal(
  cupLevel: event.record.subtype,
  earnedAt: event.record.createdAt,
)
```

Shown when the user opens the app after the Sunday calculation.

### `ProgressionService.onLevelReached` → `_onLevelReached(event)`

```
oneLiner = _levelOneLiner(event.newLevel)
emit LevelUpOverlaySignal(
  newLevel: event.newLevel,
  levelName: event.levelName,
  oneLiner: oneLiner,
)
```

**Why the one-liner lives here, not in the event:** `LevelReachedEvent` carries structural data — level number, name, previous level. Display copy (flavor text) is a presentation concern. `ProgressionService` should not know about copy. The static map lives in `EncouragementService` — the feature responsible for what the user sees and feels.

---

## Internal Functions

### `_milestoneStreakCount(subtype)` → int

Static map from `AchievementSubtype` enum to streak count.

```
threeDay    → 3
fiveDay     → 5
sevenDay    → 7
tenDay      → 10
fourteenDay → 14
```

### `_isMilestoneSubtype(subtype)` → bool

```
return subtype in {threeDay, fiveDay, sevenDay, tenDay, fourteenDay}
```

### `_levelOneLiner(level)` → String

Static map from level index to flavor text. Must match `language_guide` exactly — that doc is the authoritative source for all level copy.

```
0 → "Every master weaver was an apprentice one day."
1 → "The first threads are in place. The craft has begun."
2 → "The pattern is starting to show."
3 → "Consistent craft, recognizable work."
4 → "Mastery is taking shape, thread by thread."
5 → "Rare discipline. The loom obeys."
6 → "You tend the craft itself now."
7 → "Twenty years of discipline — thread by thread."
```

---

## Day Celebration Story Selection

### `_evaluateDayCelebration(date)`

```
dayScore = PerformanceService.getDayScore(date)
facts = AnalyticsService.computeDayFacts(date)
lastType = UserSettingsService.getLastEncouragementType()

storyType = _selectStory(facts, lastType)
storyText = _buildStoryText(storyType, facts, dayScore)

UserSettingsService.recordEncouragementSent(storyType)
emit DayCelebrationSignal(dayScore, storyType, storyText)
```

Seven story types selected by priority. Type 6 always applies as fallback. Never repeats the same type two consecutive days.

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

## Emitted Signals (Riverpod providers)

All ephemeral — consumed once and cleared. Presentation components observe these providers — never subscribe to events directly.

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
  cupLevel: AchievementSubtype
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

## Rules

- No persistent storage — `lastEncouragementType` on `UserSettingsProfile` is the only exception
- Tick subscription uses `TickGuard` — celebration fires at most once per day
- `LogFeedbackSignal` and `ThresholdMessageSignal` deferred to Phase 2
- Threshold crossings tracked in memory — fire once per crossing per session, never stored
- Signals are ephemeral — never replayed
- Presentation components observe Riverpod providers — never subscribe to events directly
- Display copy (level one-liners) lives here — not in events or Progression

---

## Dependencies

- `Heartbeat` — subscribes to `longIntervalTick`
- `ActivityService` — subscribes to `onActivityCreated`
- `CommitmentService` — subscribes to `onInstanceUpdated` where `status: closed`
- `AchievementService` — subscribes to `onAchievementEarned`
- `ProgressionService` — subscribes to `onLevelReached`
- `TickGuard` — day celebration idempotency
- `PerformanceService.getDayScore()` — day score for celebration and threshold checks
- `CommitmentIdentityService.getInstancesForDay()` — all-windows-closed check (Path A)
- `AnalyticsService.computeDayFacts()` — story selection
- `UserCoreService.getProfile()` — celebration preferences
- `UserSettingsService.recordEncouragementSent()` — story deduplication write
- `UserSettingsService.getLastEncouragementType()` — story deduplication read