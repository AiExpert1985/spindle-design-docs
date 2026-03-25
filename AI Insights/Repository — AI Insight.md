
**Created**: 15-Mar-2026
**Modified**: -
**Feature:** ai insights 
**Phase:** 3 (Firestore only — Pro/Premium users are always on Firebase)

**Availability:** Pro / Premium only.

**Purpose:** stores and retrieves the last generated AI insight per type. Minimal — three operations only. Abstract interface consistent with all other repositories.

---

## Operations

|Operation|Input|Output|
|---|---|---|
|`saveInsight`|AIInsightRecord|— overwrites previous of same type|
|`getLastInsight`|type, commitmentId?|AIInsightRecord?|
|`deleteInsight`|type, commitmentId?|—|

---

## Rules

- `saveInsight` always overwrites — never appends
- `getLastInsight` returns null if no insight exists yet for that type
- `commitmentId` only relevant for `micro_insight` type
- No pagination, no history queries — one record per type is the entire interface