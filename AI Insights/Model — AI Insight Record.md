**File Name**: model_ai_insight_record **Feature**: AI Insights **Phase**: 3 **Created**: 15-Mar-2026 **Modified**: 26-Mar-2026

---

**Purpose:** stores the last generated AI insight per type. One record per type — always overwritten when a new insight is generated. Shared between micro-insight and weekly summary components.

---

## Fields

```
AIInsightRecord
  type: InsightType          // micro_insight | quick_summary | deep_report
  definitionId: String?      // micro_insight only — identifies which commitment
  narrative: String          // full AI-generated text
  periodStart: DateTime      // start of the period analyzed
  periodEnd: DateTime        // end of the period analyzed
  generatedAt: DateTime      // when this was generated
```

```
enum InsightType { microInsight, quickSummary, deepReport }
```

`type` is an enum — the set of insight types is fixed and bounded. See `architecture_rules` §11.

---

## Why No id, createdAt, or updatedAt

This model deliberately omits the standard `id`, `createdAt`, and `updatedAt` fields. The architecture rule requiring these timestamps exists for models that accumulate records over time — where knowing when something was created vs last modified is meaningful.

`AIInsightRecord` is a single-slot model. There is always at most one record per `type + definitionId`. Every generation overwrites the previous. `generatedAt` captures the only meaningful timestamp — when this insight was produced. `id` is unnecessary because `type + definitionId` is the unique key. `updatedAt` is identical to `generatedAt` in every write, making it redundant.

---

## Storage Policy

|Type|Records kept|Key|
|---|---|---|
|microInsight|1 per commitment|type + definitionId|
|quickSummary|1 total|type|
|deepReport|1 total|type|

Maximum records at any time: number of active commitments + 2.

New generation always overwrites the previous record of the same type. No history is kept — the app tells the user explicitly that the report will be replaced on next generation.

---

## Rules

- `type + definitionId` is the unique key — no separate `id` field needed
- `definitionId` is null for `quickSummary` and `deepReport`
- No `id`, `createdAt`, or `updatedAt` — `generatedAt` is the only timestamp needed. See rationale above.
- User is responsible for saving permanently via PDF export or share — the app does not maintain history
- Written only by `AIInsightService`