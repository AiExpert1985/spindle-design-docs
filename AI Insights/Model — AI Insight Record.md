
**Created**: 15-Mar-2026
**Modified**: -
**Feature:** ai insights 
**Phase:** 3

**Purpose:** stores the last generated AI insight per type. One record per type — always overwritten when a new insight is generated. Shared between micro-insight and weekly summary components.

---

## Fields

```
type: micro_insight | quick_summary | deep_report
commitmentId: String?    // micro-insight only — identifies which commitment
narrative: String        // full AI-generated text
periodStart: DateTime    // start of the period analyzed
periodEnd: DateTime      // end of the period analyzed
generatedAt: DateTime    // when this was generated
```

---

## Storage Policy

|Type|Records kept|Key|
|---|---|---|
|micro_insight|1 per commitment|type + commitmentId|
|quick_summary|1 total|type|
|deep_report|1 total|type|

Maximum records at any time: number of active commitments + 2.

New generation always overwrites the previous record of the same type. No history is kept — the app tells the user explicitly that the report will be replaced on next generation.

---

## Rules

- No `id` field — type + commitmentId is the unique key
- No `createdAt` — `generatedAt` is sufficient
- `commitmentId` is null for quick summary and deep report
- User is responsible for saving permanently via PDF export or share — the app does not maintain history