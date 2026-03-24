
**Created**: 15-Mar-2026
**Modified**: -
**Feature**: User
**Phase:** 1 (Drift) · Phase 3 (Firestore)

**Purpose:** all data access for the user feature. Manages the single UserProfile record. Abstract interface with two implementations — local (Drift) and cloud (Firestore).

---

## Operations

| Operation | Input | Output |
|---|---|---|
| `getProfile` | — | UserProfile |
| `saveProfile` | UserProfile | — |
| `watchProfile` | — | Stream of UserProfile |

---

## Rules

- One record only — never create a second profile
- `saveProfile` is always a full replace — no partial updates
- `storageBackend` field is read by the app on startup to determine which repository implementation to activate
