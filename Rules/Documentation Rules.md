**File Name**: documentation_rules **Phase**: All phases **Created**: 21-Mar-2026 **Modified**: 21-Mar-2026

---

**Purpose:** rules for writing project documentation. Every doc should be understandable by the author after a year away from the project.

---

## 1. Standard Structure

Every doc follows this order: header → purpose → concepts → fields or functions → rules → dependencies. Skip sections with nothing to say.

---

## 2. User-Facing Before Implementation

Explain what a concept means to the user before explaining how it is implemented.

Good: "Daily — the user wants to act every day, like reading 20 pages." Bad: "Daily — generates one CommitmentInstance per calendar day with windowStart at midnight."

---

## 3. Models Describe What. Services Describe How.

Model docs explain what the object is and what its fields mean. Implementation details — how data flows, what gets generated, what events fire — belong in service docs.

Exception: a brief implementation note is allowed when it explains a model-level design decision that would otherwise be confusing.

---

## 4. Decisions Get One-Line Reasoning

State the reason for a design choice in one sentence. If the reasoning is already in `architecture_rules`, reference it rather than repeating it.

Good: "`commitmentType` is immutable — changing it would make all historical performance meaningless."

---

## 5. Fields: List First, Explanations Below

Show the field list as a code block, then explain each field below. One or two sentences per field: what it is, how the user sees it, any rules that govern it.

---

## 6. Examples Must Be Self-Contained

An example should make sense without knowing the project. Use plain language, not class names.

Good: "For example, 'Read 200 pages this week' is a weekly Do commitment with target 200 and unit pages." Bad: "For example, `CommitmentDefinition` with `recurrenceType: weekly`, `target.value: 200`."

---

## 7. Be Concise

Add detail only when it prevents misunderstanding. If a sentence can be removed without losing meaning, remove it.

---

## 8. No Cross-Doc Repetition

Reference other docs rather than repeating their content. Repetition creates maintenance debt.

---

## 9. Write for Your Future Self

Assume the reader has not seen this project for a year. Use plain language. Define any term that is not obvious from context.

---

## 10. Diagrams Are Views, Not Sources

`feature_dependency_diagram.puml` is a visual summary of `feature_dependency_chain.md`. When they conflict, the text doc is correct. When the diagram looks stale, ask Claude to regenerate it from `feature_dependency_chain.md` — do not edit the diagram manually.

---

## 11. Model Docs Describe Data Only

A model doc states what fields exist, what they mean, and what data rules apply (immutability, valid ranges, required vs optional). It never mentions how the UI presents or restricts the data, and never mentions which service calls what or how.

Good: "`value` must be > 0" Bad: "the input field is disabled until value > 0" or "ActivityService validates this before writing"

UI behaviour belongs in component docs. Service behaviour belongs in service docs.

---

## 12. Components State Their Expected Placement

Every component doc that produces visible UI must include a brief note on where it is expected to appear in the app. Describe the location by purpose, not by exact screen name — screen names change, purposes do not.

Good: "expected to appear in the screen that shows the full detail of one commitment" Bad: "placed in CommitmentDetailScreen"

This is not a binding contract — placement may change. It gives any developer reading the doc an immediate sense of context without requiring them to trace through the full navigation structure.

---

## 13. Feature Dependency Diagram Rules

When generating or updating `feature_dependency_diagram.puml`, follow these rules.

**Layout**

- Features ordered top to bottom by dependency level — foundation at top, highest-level at bottom
- Same-level features (e.g. Garment, Rewards, Encouragement) placed side by side, each in its own box
- Each feature is a tall rectangle with the feature name at the top

**Content**

- Inside each feature box: small boxes for public functions and events accessed by other features only — not internal functions, not private methods
- Show functions in one row where possible
- Exclude infrastructure services (EventBus, TickService, TemporalHelper), repositories, UI components, and Riverpod providers

**Arrows**

- Arrows point downward from the caller or subscriber to the function or event it accesses
- Solid arrow (`-->`) = direct service call
- Dashed arrow (`..>`) = event subscription
- Each arrow labeled with the function name or event name

**Scope**

- Only show connections between features — no self-arrows, no intra-feature calls
- Only include public functions actually called by another feature — omit unused public functions