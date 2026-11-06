# Inventory · Membership eligibility (example)

> Illustrative example. Program, table and file names are made up to show how the template is filled in.

Seam: `CLMADJ00` → `CALL 'ELGB100' USING ELGCOMM`
Discovered: 2026-10-14 by Cartographer

## Programs

| Program | Role | Kind | Lines | Paragraphs in scope | Notes |
|---|---|---|---|---|---|
| ELGB100 | Eligibility driver | CICS online, trans EL01 | 2,410 | 9 | Entry point from the seam |
| ELGB110 | Member lookup and cross-reference | Called | 1,180 | 7 | Reads VSAM MBRXREF |
| ELGB120 | Coverage span resolver | Called | 1,960 | 12 | Cursor reads WITH UR |
| ELGB130 | Plan and benefit attachment | Called | 870 | 6 | |
| ELGB140 | Retro-term and grace period | Called | 1,040 | 8 | One undocumented rule (R-03) |
| ELGB150 | Dependent age-out | Called | 520 | 4 | |
| ELGB900 | Response builder | Called | 410 | 5 | Builds the 271 EB segment |

## Copybooks

| Copybook | Used by | Library / version | Layer |
|---|---|---|---|
| ELGCOMM | ELGB100, ELGB900 | repo | L0 |
| MBRMAST | ELGB110, ELGB140, ELGB150 | repo | L1, L2 |
| CVGSPAN | ELGB120 | repo | L2 |
| GRPPLAN | ELGB130 | repo | L5 |
| ELGRSP | ELGB100, ELGB900 | repo | L5 |
| DATECONV | ELGB120, ELGB140 | SYS2.COPYLIB, fetched via Endevor (R-04) | L6 |

## Relational tables (L2)

| Table | Statements | Read or write | Isolation | Used by |
|---|---|---|---|---|
| MBR_MASTER | 4 | Read | CS | ELGB100, ELGB110, ELGB140 |
| MBR_COVG_SPAN | 5 | Read | UR | ELGB120 |
| GRP_PLAN_XREF | 3 | Read | CS | ELGB130 |
| PLAN_BNFT_PKG | 2 | Read | CS | ELGB130 |

## Keyed files (L3)

| File | Type | Key | Access path today | Used by |
|---|---|---|---|---|
| MBRXREF | VSAM KSDS | Alternate (card) ID | CICS READ only | ELGB110 |
| ELGHIST | VSAM ESDS | None (append) | CICS WRITE | ELGB100 audit |

## Batch jobs that touch the same data (L4)

| Job | Schedule | Writes to | Overlap risk |
|---|---|---|---|
| ELGJ834 | Nightly 01:30–03:10 | MBR_MASTER, MBR_COVG_SPAN | Online reads WITH UR can see half-loaded enrollment (R-07) |

## Out of scope

| Program or paragraph | Why | Which future wave |
|---|---|---|
| ELGB160 COB primacy indicator (10 paragraphs) | Belongs to coordination of benefits | Wave 4, COB |
