# Decisions · Membership eligibility (example)

> Illustrative example. Names are roles, not people.

### D-01 · G1 for units not held by open items

| Field | Value |
|---|---|
| Type | gate G1 |
| Scope | 48 of 51 units (all except those held by R-01, R-02, R-03) |
| Decided by | Eligibility SME |
| Date | 2026-10-24 |

**Decision**
Plain-language rules confirmed for the listed units in `units.json` where `g1` is `D-01`.

**Reason**
Rules match current policy and the golden cases reviewed.

### D-02 · Masking specification for golden records

| Field | Value |
|---|---|
| Type | behavior change (test data only) |
| Scope | All golden records in every wave |
| Decided by | Privacy officer |
| Date | 2026-10-28 |

**Decision**
Deterministic tokens for member IDs, names and card IDs. Dates shifted by a per-member offset so intervals and month boundaries hold. Records failing the check are discarded.

**Reason**
Golden records come from production claims.
