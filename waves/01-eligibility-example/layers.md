# Data layers · Membership eligibility (example)

> Illustrative example.

| Layer | What this wave found | TypeScript side | Research read path | Runtime read path | Status | Items |
|---|---|---|---|---|---|---|
| L0 Contract | ELGCOMM, 312 bytes, 41 fields, 3 REDEFINES | `EligibilityRequest` / `EligibilityResponse` plus byte-exact codec | Captured at the seam | Seam | Resolved | |
| L1 Business rules | 51 paragraphs in scope, 1 rule with no spec | Pure functions under `rules/`, `coverage/`, `member/` | Source | n/a | Needs a person | R-03 |
| L2 Relational data | 14 SQL statements, 2 cursors WITH UR | Read-only adapters behind interfaces | Masked replica ELGRPL (pending) | Replica endpoint | Needs a person | R-01, R-07 |
| L3 Keyed files | MBRXREF has no query path outside CICS | `memberXref` adapter | Nightly unload (proposed) | CICS web service EL0X (proposed) | Blocked | R-02 |
| L4 Batch feeds | ELGJ834 writes spans nightly 01:30–03:10 | Not migrated; freshness contract | Job schedule | n/a | Decision pending | R-07 |
| L5 Reference codes | 62 status and reason codes, 3 product lines | Generated enums keeping COBOL values | Copybooks | Bundled | Resolved | |
| L6 Representation | Missing DATECONV, EBCDIC sort, COMP-3 premiums | `ebcdicCompare()`, integer cents, nullable dates | Source | Shared helpers | Resolved | R-04, R-05, R-06 |
