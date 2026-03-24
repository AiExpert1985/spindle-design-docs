**File Name**: model_bonus_thread_record **Feature**: Progression **Phase**: 3 **Created**: 24-Mar-2026 **Modified**: 24-Mar-2026

---

**Purpose:** permanent record of every bonus thread awarded to the user. Written once at award time — never edited, never deleted. Used by the Progression screen for bonus history display and by `ProgressionService` for trigger evaluation.

---

## Fields

```
BonusThreadRecord
  id: String
  weekStart: DateTime      // the week this bonus was awarded for
  bonusAmount: double      // 1.0 or 2.0 — see bonus trigger table in AppConfig
  trigger: BonusTrigger    // which condition fired
  occurredAt: DateTime     // when the bonus was awarded
  createdAt: DateTime
```

```
enum BonusTrigger {
  firstCup
  firstDiamond
  threeConsecutive
  comeback
  diamondComeback
}
```

---

## Rules

- Append-only — never edited or deleted
- Written by `ProgressionService` before `BonusThreadAwardedEvent` is published — record exists before any subscriber reacts
- `weekStart` identifies which week triggered the bonus — used for deduplication
- One record per trigger event — never multiple records for the same week and trigger
