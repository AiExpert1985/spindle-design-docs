**File Name**: commitmentinstancerepository **Feature**: Commitment **Phase**: 1 **Created**: 15-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** storage interface for CommitmentInstance records. Called only by CommitmentIdentityService. PerformanceService writes `livePerformance` indirectly — it publishes `LivePerformanceCalculatedEvent`, CommitmentIdentityService receives it and calls `updateLivePerformance()` which writes here. No other feature accesses this repository directly.

---

## Operations

- `saveInstance(instance)` — upsert. Sets `updatedAt` on every call.
- `getInstance(id)` — returns one instance by ID.
- `getInstanceForCommitmentOnDate(definitionId, date)` — returns the instance whose window contains the given date for the given commitment. Used by PerformanceService (via CommitmentIdentityService) to identify which instance a log belongs to.
- `watchInstancesForDay(date)` — stream of all instances whose window contains the given date. Emits on any change. Used by dashboard and commitment cards.
- `getInstancesForPeriod(from, to, definitionId?)` — instances in a date range, optionally filtered by commitment. Used by Analytics, performance calendar, Your Record, and PerformanceService query functions.
- `getPendingInstancesForCommitment(definitionId)` — all pending instances for one commitment. Used when clearing before recreation.
- `updateLivePerformance(instanceId, value)` — updates `livePerformance` and `updatedAt` only. Narrow write path — avoids a full instance save when only livePerformance changes.
- `deleteInstance(id)` — hard delete for a single instance.
- `deletePendingInstancesForCommitment(definitionId)` — deletes all pending instances. Closed instances untouched.
- `deleteAllInstancesForCommitment(definitionId)` — deletes all instances including closed history. Only on permanent commitment deletion.

---

## Rules

- Called only by CommitmentIdentityService
- No business logic — only fetch and store
- All IDs are client-generated UUIDs
- `deleteAllInstancesForCommitment` is irreversible — only reachable through the permanent deletion path