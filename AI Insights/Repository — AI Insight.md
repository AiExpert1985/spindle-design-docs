**File Name**: repository_ai_insight **Feature**: AI Insights **Phase**: 3 (Firestore only — Pro/Premium users are always on Firestore) **Created**: 15-Mar-2026 **Modified**: 28-Mar-2026

**Availability:** Pro / Premium only.

---

**Purpose:** stores and retrieves the last generated AI insight per type. Minimal — three operations only. Called only by `AIInsightService`.

---

## Interface

```dart
abstract class AIInsightRepository {
  Future<void> saveInsight(AIInsightRecord record);
  Future<AIInsightRecord?> getLastInsight(InsightType type, {String? definitionId});
  Future<void> deleteInsight(InsightType type, {String? definitionId});
}
```

`saveInsight` — always overwrites the previous record of the same `type + definitionId`. Never appends.

`getLastInsight` — returns null if no insight exists yet for that type. `definitionId` only relevant for `microInsight` type.

`deleteInsight` — removes the record. Used when a commitment is permanently deleted.

---

## Rules

- Called only by `AIInsightService`
- `saveInsight` always overwrites — never appends
- No pagination, no history queries — one record per type is the entire interface
- Firestore only — this repository has no Drift implementation. AI Insights is Pro/Premium, which always uses Firestore.