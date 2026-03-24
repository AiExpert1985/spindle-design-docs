**Created**: 15-Mar-2026 **Modified**: 18-Mar-2026 **Feature**: Rewards **Phase**: 2 **Standalone:** yes — can be added or removed without touching other components.

**Purpose:** rewards consistent weekly performance with a collectible cup. One cup per week, earned when the weekly performance ratio meets a threshold. Permanent record.

---

## How Cups Are Earned

Cups are based on the **weekly performance ratio** — not a raw percentage score. The ratio measures how close the user came to their target across all commitments, normalized so the result is fair regardless of how many commitments they have.

|Cup|Weekly performance ratio|Meaning|
|---|---|---|
|🥉 Bronze|≥ 0.60|60% of daily target on average|
|🥈 Silver|≥ 0.75|75% of target — solid week|
|🥇 Gold|≥ 0.85|85% of target — strong week|
|💎 Diamond|≥ 0.95|Near-perfect — streaks and bonuses firing|

Below 0.60 — no cup that week. No record written.

The ratio is calculated by `PerformanceAccountingService` and published as `WeeklyPerformanceCalculatedEvent`. `RewardService` subscribes and determines cup level.

---

## UI — Your Record (Header)

```
🥉   🥈   🥇   💎
 8    5    3    1
```

- One row, four cups, count inside each
- Diamond cup is visually larger — signals rarity
- Tap any cup → scrollable popup listing weeks earned at that level:

```
🥇 Gold cups (3)
• Week of Nov 25 — ratio 0.87
• Week of Nov 4  — ratio 0.91
• Week of Oct 14 — ratio 0.86
```

---

## Data Model

One record per week a cup is earned:

```
id
weekStart        Monday 00:00:00 of that week
cupLevel         bronze | silver | gold | diamond
weeklyRatio      double    // the performance ratio that earned this cup
pointValue       double    // 1, 2, 3, or 5 — stored at calculation time
createdAt
```

---

## Trigger

`PerformanceAccountingService` publishes `WeeklyPerformanceCalculatedEvent` on Sunday night after sealing the week. `RewardService` subscribes and calls `processWeeklyCup(weekStart, ratio)`.

---

## Notification

`RewardService` publishes `CupEarnedEvent`. `NotificationService` subscribes and sends: `"You earned a silver cup this week."` → deep-links to `/record?section=cups`

---

## Dependencies

- `RewardService` — cup calculation, event subscription and publication
- `PerformanceAccountingService` — publishes `WeeklyPerformanceCalculatedEvent`
- `NotificationService` — subscribes to `CupEarnedEvent`, sends notification
- `EventBus` — event communication