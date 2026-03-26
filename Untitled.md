Perfect. That's the cleanest version. Let me confirm:

---

**The design:**

```dart
// Lives in Infrastructure
abstract class Achievable {
  AchievementRecord toAchievementRecord();
}
```

Each feature's **result object** (not the event itself) implements `Achievable`:

```dart
// Inside Cups feature
class WeeklyCup implements Achievable {
  // ... cup fields ...
  
  @override
  AchievementRecord toAchievementRecord() {
    return AchievementRecord(
      type: AchievementType.cup,
      subtype: cupLevel.name,
      sourceId: id,
      definitionId: null,
      earnedAt: createdAt,
    );
  }
}
```

Each feature still publishes its own event (`CupEarnedEvent`, `StreakChangedEvent`, `RewardEarnedEvent`) carrying its own domain model. `AchievementService` subscribes to those events, and when one arrives calls `event.source.toAchievementRecord()` — or more precisely, the model inside the event implements `Achievable`.

---

**The flow:**

```
CupService earns a cup
  → creates WeeklyCup (implements Achievable)
  → saves to CupRepository
  → publishes CupEarnedEvent(cup: WeeklyCup)

AchievementService subscribes to CupEarnedEvent
  → calls event.cup.toAchievementRecord()
  → saves AchievementRecord
  → publishes AchievementEarnedEvent(record)
```

No parsing. No switch statements. No registration. Compile-time enforced. Each feature decides its own event shape — the only contract is that the model carried by the event implements `toAchievementRecord()`.

---

**Where `AchievementRecord` and `AchievementType` live:**

They must live in **Infrastructure** — so that `WeeklyCup`, `StreakRecord`, `RewardRecord` can reference them in `toAchievementRecord()` without creating upward dependencies. Since these are shared contracts consumed across the whole app, Infrastructure is the right home.

---

**Is it compatible with our architecture?**

Fully. Each feature:

- Owns its domain model and event — no change
- Implements one interface method — minimal addition
- Publishes its own event — no change
- Never knows about `AchievementService` — no upward coupling

`AchievementService`:

- Subscribes to events from below — same pattern as everywhere else
- Calls `toAchievementRecord()` — one line, fully typed
- Never imports feature internals — only knows `Achievable` and `AchievementEarnedEvent`

---

**What moves to Infrastructure:**

```
Achievable interface       → toAchievementRecord()
AchievementRecord model    → shared contract
AchievementType enum       → shared contract
AchievementEarnedEvent     → published by AchievementService
```

Everything else stays where it belongs — in each feature.

---

This is clean, consistent, enforced, and extensible. Shall I now update the Feature Dependency Chain and all affected docs?