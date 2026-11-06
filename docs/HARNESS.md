# Legacy Carve-out Harness: the model

Version 1.0 · 2026-11-05

This document is the source of truth for moving a function out of a COBOL mainframe system into TypeScript, one function at a time. It is written for the people running the migration and for the AI agents doing the work. When this document and any other instruction disagree, this document wins unless a gate owner records an exception in the wave's `decisions.md`.

The model has two halves:

1. **Carve-out** (this document): a one-time migration per function, from discovery to cutover.
2. **Runtime** ([`runtime/`](runtime/)): how the harness uses the migrated TypeScript functions, and any remaining COBOL functions, to process claims once a function is live.

---

## 1. Rules that never bend

1. **One seam at a time.** A wave starts at a single existing CALL boundary in the legacy system. The harness never migrates code that is not reachable from that seam.
2. **Research before conversion.** A unit is not converted until its research items are resolved or waived by their owner.
3. **The mainframe defines correct.** Until cutover, the TypeScript is correct only when it produces the same output as COBOL on the same input. Any intended behavior change is a recorded decision. It is never a silent improvement.
4. **Diffs become rules.** A parity diff is never fixed by hand in the TypeScript alone. It becomes a ledger rule, the converter is re-run for every unit the rule touches, and parity is measured again.
5. **COBOL keeps answering until G3.** Before gate G3, the COBOL answer is the one returned to the caller. After cutover, COBOL stays deployed as a fallback for 90 days.
6. **No unmasked member data.** Golden records are masked at capture. The harness refuses to store a record that fails the masking check.
7. **Two-way traceability.** Every TypeScript function names the COBOL paragraphs it came from. Every paragraph names the TypeScript it became, or the reason it was not migrated.

---

## 2. The seam

```
caller (unchanged)  ──►  routing seam (new)  ──►  legacy program (COBOL)
                                        └──►  modern module (TypeScript)
```

- The seam accepts and returns the **exact** legacy interface: the same COMMAREA or linkage record, byte for byte.
- The seam has one mode per traffic segment (for example, by line of business or group type):

| Mode | Who answers | TypeScript runs? | Use |
|---|---|---|---|
| `off` | COBOL | No | Before harvesting starts |
| `shadow` | COBOL | Yes, the result is compared and discarded | Verify and shadow phases |
| `canary` | TypeScript for N% of the segment | Yes | After G3, first segments |
| `primary` | TypeScript | COBOL kept as fallback | Cutover |

- The seam records every call's input, output and the rows it read. This record is the source of golden test data.

---

## 3. Units and stages

A **unit** is one COBOL paragraph, or a PERFORM THRU range treated as one, in scope for the wave. Units move independently. One blocked unit does not stop the rest of the module.

| Stage | Name | A unit enters this stage when |
|---|---|---|
| 0 | Discovered | It is reachable from the seam and listed in `inventory.md` |
| 1 | Researched | It has a plain-language rule, a data-layer tag and a TypeScript target, and no open research item holds it |
| 2 | Tests harvested | It has golden cases covering every branch it owns |
| 3 | Converted | The TypeScript compiles and its unit tests pass |
| 4 | Parity verified | Every golden case matches COBOL at this unit's step, or the diff is explained by a ledger rule |
| 5 | In shadow | It runs in production in shadow mode with no unexplained diff for the shadow window |

A unit **held** by an open research item stays at stage 1 at most. It is shown with a red outline in the console.

---

## 4. Phases

Phases run left to right. A wave is "in" several phases at once because units move independently. The wave is only *through* a phase when every in-scope unit is.

| # | Phase | Agent | Reads | Produces | Exit |
|---|---|---|---|---|---|
| 01 | Intake | Harness | A function name from a person | Seam, scope boundary, wave folder | Function owner confirms the scope |
| 02 | Discover | Cartographer | Source repo, change-management listings (Endevor or ChangeMan), JCL, copybook libraries | `inventory.md`: programs, paragraphs, copybooks, tables, files, jobs | Every COPY and CALL resolves to a file in the repo |
| 03 | Research | Archaeologist, Data steward | Each paragraph and each data touch | Plain-language rules, `layers.md`, `research-plan.md` | **G1** |
| 04 | Harvest tests | Test harvester | Masked records captured at the seam, replica reads | `golden/` cases per unit | Every branch of every unit has at least 3 cases |
| 05 | Convert | Translator | Rules, copybook-derived types, `ledger.md` | TypeScript module and unit tests | Compiles, unit tests pass |
| 06 | Verify | Verifier | Golden set through both engines | `parity/` step-level diff reports | **G2** |
| 07 | Shadow | Verifier | Live traffic, COBOL still answering | Daily diff report | **G3** |
| 08 | Cutover | Router | Segment plan | Segments moved to `canary`, then `primary` | 90 days with no fallback use, then COBOL retired |

---

## 5. Gates

Gates are the only places where a person must approve before work continues. Each sign-off is recorded in the wave's `decisions.md` with name, date and scope.

| Gate | After | Signed by | Passes when |
|---|---|---|---|
| **G1** Research sign-off | Research | Function SME, and the owner of each research item | Every rule is confirmed in plain language. Every research item is resolved or waived. Every data layer has a read path for tests. |
| **G2** Parity | Verify | Solution architect | Parity on the golden set is at least **99.90%**, and every remaining diff is explained by a ledger rule or a recorded decision |
| **G3** Cutover approval | Shadow | Release manager | 30 consecutive days in shadow with no unexplained diff, plus a rollback rehearsal completed |

G1 is granted **per unit**. A unit can pass G1 while others are still held.

---

## 6. Agents

| Agent | Does | Must not |
|---|---|---|
| Cartographer | Walks CALL, COPY, EXEC SQL, EXEC CICS, file SELECT/ASSIGN and JCL DD statements from the seam | Guess at a missing member; it opens a research item instead |
| Archaeologist | Writes each paragraph as a rule a business person can read, and flags rules with no documented source | Describe intent it cannot see in code or documents |
| Data steward | Tags every data touch with a layer (§8) and finds the read path for tests and for runtime | Ask for production write access, ever |
| Test harvester | Captures masked inputs, outputs and rows at the seam and picks cases per branch | Store anything that fails the masking check |
| Translator | Writes TypeScript per unit from the rule, the copybook types and the ledger | Fix a parity diff without a ledger rule |
| Verifier | Runs both engines, diffs at every step, and traces each diff to a step and a cause | Mark a diff explained without naming the rule or decision |
| Librarian | Writes and maintains ledger rules, and re-queues every unit a rule touches | Adopt a rule that changes behavior without a decision record |

## 7. People

| Role | Decides |
|---|---|
| Function SME | Whether an extracted rule is intended (G1) |
| Mainframe DBA and systems | Read access to DB2 replicas, VSAM files and job schedules |
| Solution architect | Accepted behavior changes (for example, read isolation), and G2 |
| Privacy officer | The masking specification for golden data |
| Release manager | Segment order for cutover, and G3 |

---

## 8. Data layers

Every unit touches one or more layers. A layer is **resolved** when the harness can (a) read it during research, (b) describe it as TypeScript types, and (c) reach it at runtime.

| Layer | Name | Detect it by | TypeScript side | Usual resolution |
|---|---|---|---|---|
| L0 | Contract | `LINKAGE SECTION`, `CALL … USING`, COMMAREA copybooks | Request/response types plus a byte-exact codec | Auto: the codec must round-trip every captured record |
| L1 | Business rules | Paragraphs, `PERFORM … THRU`, 88-level conditions, `EVALUATE` | Pure functions grouped by rule family | Auto extraction; SME confirms in G1 |
| L2 | Relational data | `EXEC SQL`, `DECLARE CURSOR`, isolation clauses such as `WITH UR` | Read-only adapter behind an interface | API: read access on a masked replica, then an endpoint |
| L3 | Keyed files | `SELECT … ASSIGN`, `EXEC CICS READ`, VSAM KSDS/ESDS, IMS segments | Adapter per file | Person: a CICS web service for runtime, or an unload for tests |
| L4 | Batch feeds | JCL jobs that write to the same tables or files, plus their schedule | Not migrated in the wave; freshness contract documented | Person: decision on reads during load windows |
| L5 | Reference codes | 88-levels, code tables, hard-coded literals | Generated enums keeping the COBOL value | Auto: regenerated on every copybook change |
| L6 | Representation | EBCDIC, `COMP-3`, `COMP`, century-coded dates, `LOW-VALUES`/`HIGH-VALUES`, open-ended sentinels | Shared helpers: `ebcdicCompare()`, integer-cent money, date types with null for open end | Auto; each finding becomes a ledger rule |

---

## 9. Research items

Anything the harness cannot resolve alone becomes a research item in `research-plan.md`, using [`templates/research-item.md`](../templates/research-item.md).

- **Kinds:** `auto` (the harness resolves it), `api` (needs an endpoint or access, then the harness calls it), `human` (needs a decision or confirmation).
- **Statuses:** `open` → `waiting` (on a named person) → `resolved` or `waived`. A waiver needs the owner's name and a reason in `decisions.md`.
- Every item names the units it **holds**. Units not held keep moving.
- An `api` item always includes the exact request the harness will make, read-only, against a masked source.

---

## 10. Tests and parity

- **Golden set:** masked records captured at the seam, with at least 3 cases per branch per unit. Include the edge cases research found: retro changes, voids, open-ended dates, maximum table sizes, and calls made during batch windows.
- **Step-level diff:** compare the output of each paragraph or function, not only the final answer. The first differing step is where the cause is.
- **Replay:** until L2 and L3 have live read paths, the TypeScript side replays the rows the seam captured.
- **Parity rate** = matching cases ÷ total cases on the golden set, per build. G2 needs 99.90% with every remaining diff explained.

---

## 11. The learning ledger

`ledger.md` sits at the repo root and is shared by every wave. Each rule uses [`templates/ledger-rule.md`](../templates/ledger-rule.md).

- **Sources:** a research finding or a parity diff.
- **Lifecycle:** `proposed` → `adopted` → `retired`. Only adopted rules feed the Translator.
- A rule that changes behavior compared with COBOL also needs a decision record before adoption.
- When a rule is adopted, the Librarian lists every unit in every active wave that the rule touches and moves them back to stage 3 for re-conversion.

---

## 12. Repository layout for a project using this model

```
docs/HARNESS.md              this document
CLAUDE.md                    tells Claude Code to follow this document
ledger.md                    learning ledger shared by all waves
waves/
  NN-<function>/
    inventory.md             phase 02 output
    layers.md                data layers for this wave
    research-plan.md         research items
    units.json               every unit, its stage, its rule and target
    decisions.md             gate sign-offs, waivers and behavior changes
    golden/                  masked golden cases (never committed unmasked)
    parity/                  per-build diff reports
```

## 13. Choosing the next wave

Score each candidate function:

1. **Seam clarity:** a single CALL with a stable interface scores highest.
2. **Dependencies:** prefer functions whose callees have already been migrated. A function depending on one still in COBOL waits until that one reaches shadow.
3. **Data reach:** fewer L3 keyed files and fewer L4 batch overlaps mean less waiting on people.
4. **Ledger coverage:** a higher share of its L6 patterns already in the ledger means less new research.

Example order for a claims adjudication engine: membership eligibility → provider network status → benefit accumulators → coordination of benefits → prior authorization match → claim edits → pricing → payment and remittance.

---

## 14. Glossary

- **COMMAREA:** the fixed-layout record passed between CICS programs.
- **Copybook:** a shared COBOL source fragment, usually a record layout, pulled in with `COPY`.
- **Paragraph:** a named block of COBOL procedure code; the unit of migration here.
- **PERFORM THRU:** runs a range of paragraphs in order; it can fall through.
- **COMP-3:** packed decimal, two digits per byte with a sign nibble.
- **EBCDIC:** mainframe character encoding. Its sort order puts letters before digits.
- **CYYMMDD:** a date where C is the century flag (0 = 19xx, 1 = 20xx).
- **KSDS / ESDS:** VSAM keyed and entry-sequenced files.
- **834 / 270 / 271:** X12 transactions for enrollment, eligibility inquiry and eligibility response.
- **Golden case:** a masked, captured input with the COBOL output it produced.
