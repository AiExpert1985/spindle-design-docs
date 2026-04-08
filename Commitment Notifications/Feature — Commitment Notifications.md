**File Name**: feature_commitment_notifications **Phase**: 1 **Created**: 28-Mar-2026 **Modified**: 28-Mar-2026

---

CommitmentNotifications watches commitment windows and fires timely notifications to keep the user aware of approaching and closing windows — without knowing anything about performance, scoring, or what the user actually logged.

---

## Why It Exists

Commitment owns instances and their windows. Infrastructure owns raw notification delivery. Neither should own the logic of _when_ to notify the user about a commitment — that requires knowing about window boundaries, warning offsets, and commitment state, which is domain knowledge that belongs in its own dedicated place.

CommitmentNotifications bridges this gap. It is the only feature that combines knowledge of commitment windows with the ability to send notifications. It is fully removable — deleting it stops notifications, nothing else is affected.

---

## Position in the System

Sits above Commitment and Activity, and uses Infrastructure and TemporalHelper. Depends on Commitment for instance events, Activity to check whether the user already logged before firing, Heartbeat for the short-interval tick, NotificationService for delivery, and TemporalHelper to format human-readable lead times in message text.

Consuming only — it publishes no events. Its entire output is notifications delivered to the user's device.

---

## How It Works

The feature maintains a small set of pending notification records. Each record holds the message to send, the route to navigate to on tap, and the timestamp when it should fire. The feature has two responsibilities: keeping those records current, and firing them on time.

**Keeping records current.** Whenever a commitment instance changes — created, updated, or deleted — the feature clears all pending records for that commitment and re-evaluates from scratch. Two records are written per active instance: a warning notification that fires before the window closes, and a check-in notification that fires at window end. If the commitment is frozen, completed, or deleted, no records are written.

This means state changes are handled automatically and for free. When a commitment is frozen, completed, or soft-deleted, an `InstanceUpdatedEvent` arrives just like any other change. The service clears all pending records for that commitment, re-evaluates, finds the commitment is not active, and writes nothing. The notifications are gone. No special case for deactivation — the same path handles everything.

The replace-all approach on every instance change is deliberate. Patching individual records when specific fields change would require tracking every possible transition — which field changed, whether the timing was affected. Replace-all is simpler and always correct: whatever changed, the records are fresh.

**Firing on time.** On every short-interval tick, the feature queries for all records whose timestamp is at or before the current time. For each due record, it checks whether the user already logged activity for that commitment on that window's date. If they have, the record is deleted silently — no notification sent. If they have not, the notification fires and the record is deleted. Checking at fire time against the current state of the log handles the edge case where the user logged and then deleted the entry — the check correctly finds no activity and fires the reminder.

**Deterministic record IDs.** Each record's ID is constructed from the commitment ID, window date, and notification type. This means re-scheduling a notification is always a replace, never a duplicate insert. No cleanup of old records needed on reschedule.

**What it does not do.** It never reads activity logs, never checks performance, never knows whether the user completed a commitment. It only knows when windows open and close and whether a commitment is active.

---

## Events

CommitmentNotifications publishes no events. It is a pure consumer — it subscribes to instance events from Commitment and tick signals from Heartbeat, and its only output is notifications pushed to the user's device.

---

## Rules

- Never fires for frozen, completed, or deleted commitments
- On any instance change: clear all records for that commitment, then re-evaluate — never patch
- Record deletion after firing is the only idempotency needed
- Message text is fully constructed here — NotificationService receives final strings only
- Record IDs are deterministic — constructed from `definitionId`, window date, and notification type
- Never calls any scoring or performance feature

---

## Later Improvements

**Grace period.** Gives the user a short extra window to log after their commitment window closes. The user triggers it from the activity log screen. Same mechanism, same repository — one additional write path triggered by user action. Deferred until the activity log screen UI decisions are made.


---

## Related Docs

[[Model — Commitment Notification]]
[[Repository — Commitment Notification]]
[[Service — Commitment Notification]]
