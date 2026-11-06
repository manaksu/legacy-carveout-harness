# Research plan · Membership eligibility (example)

> Illustrative example. Research ends when every item is resolved or waived. Units not held by an open item keep moving.

| Item | Title | Layer | Kind | Owner | Status | Holds |
|---|---|---|---|---|---|---|
| R-01 | Read access to coverage spans | L2 | api | Mainframe DBA | waiting | ELGB120 1100-FETCH-SPANS |
| R-02 | No query path into MBRXREF | L3 | human | Mainframe systems | waiting | ELGB110 1100-XREF-ALT-ID |
| R-03 | Undocumented retro-term timing rule | L1 | human | Eligibility SME | waiting | ELGB140 1100-POSTED-VS-EFFECTIVE |
| R-04 | Copybook DATECONV missing from repo | L6 | auto | Harness | resolved | |
| R-05 | EBCDIC order in span sort | L6 | auto | Harness | resolved | |
| R-06 | COMP-3 premium balances | L6 | auto | Harness | resolved | |
| R-07 | Uncommitted reads during 834 load | L2, L4 | human | Solution architect | waiting | Gate G2 |
| R-08 | Member data in golden records | All | human | Privacy officer | resolved | |

---

### R-01 · Read access to coverage spans for test harvesting

| Field | Value |
|---|---|
| Layer | L2 |
| Kind | api |
| Owner | Mainframe DBA |
| Status | waiting |
| Holds | `ELGB120 1100-FETCH-SPANS` |
| Opened | 2026-10-16 |

**Why it matters**
The harness can replay spans captured at the seam, but it needs spans for any member to build edge cases: voids, reinstatements and members with 40 spans.

**Action plan**
1. DBA grants SELECT on masked replica schema ELGRPL to the harness service ID.
2. Harness calls the endpoint below for each golden member and stores rows with the test.
3. Harness re-runs harvesting for ELGB120 and moves 1100-FETCH-SPANS to stage 2.

**API call**
```
GET /elig-data/v1/members/{mbrId}/coverage-spans?asOf=2026-03-14
  read-only, masked replica ELGRPL, via z/OS Connect
  returns: [{ planCd, effDt, termDt, status, postedTs }]
```

### R-02 · No query path into the MBRXREF keyed file

| Field | Value |
|---|---|
| Layer | L3 |
| Kind | human |
| Owner | Mainframe systems |
| Status | waiting |
| Holds | `ELGB110 1100-XREF-ALT-ID` |
| Opened | 2026-10-16 |

**Why it matters**
Card IDs on claims resolve to members only through VSAM. Nothing outside CICS can read it today.

**Action plan**
1. Option A: wrap a CICS READ in web service EL0X for runtime (about 2 weeks).
2. Option B: nightly REPRO unload to a file the harness loads for tests (about 2 days).
3. Harness recommends B now for research and A before canary.

**API call**
```
Option A: GET /elig-data/v1/xref/{altId}  ->  { mbrId, sfx }
Option B: REPRO INFILE(MBRXREF) OUTFILE(XREFUNL), nightly 03:30
```

### R-03 · Undocumented rule in retro-term timing

| Field | Value |
|---|---|
| Layer | L1 |
| Kind | human |
| Owner | Eligibility SME |
| Status | waiting |
| Holds | `ELGB140 1100-POSTED-VS-EFFECTIVE` |
| Opened | 2026-10-18 |

**Why it matters**
COBOL ignores a retro term posted within 3 business days of its term date. No spec or ticket explains it. It changes the outcome for about 0.4% of termed members.

**Action plan**
1. Harness shows the SME the paragraph, the plain-language rule and the 12 golden claims it affects.
2. SME confirms it is intended, or marks it as a defect to keep for parity and fix after cutover.
3. Harness records the answer as a ledger rule and converts the unit.

### R-07 · Uncommitted reads during the 834 load

| Field | Value |
|---|---|
| Layer | L2, L4 |
| Kind | human |
| Owner | Solution architect |
| Status | waiting |
| Holds | Gate G2 |
| Opened | 2026-10-29 |

**Why it matters**
COBOL reads spans WITH UR, so a claim adjudicated between 01:30 and 03:10 can see a half-loaded 834. The replica serves committed rows only. 4 of the 5 open parity diffs trace to this.

**Action plan**
1. Architect accepts read-committed as the new behavior, or asks for the load window to be fenced.
2. Harness marks the 4 diffs as explained under that decision.
3. Gate G2 is re-evaluated.

### R-04, R-05, R-06, R-08 · Resolved

- **R-04** DATECONV found in SYS2.COPYLIB through the Endevor listing and added to the repo mirror with its version stamp. Produced L-012.
- **R-05** Parity diff on sample S3 showed ASCII ordering of plan codes. Added `ebcdicCompare()`. Produced L-013.
- **R-06** Paid-through dates derived from COMP-3 balances drifted by a cent in floating point. Produced L-011.
- **R-08** Masking specification approved 2026-10-28: deterministic tokens for IDs and names; dates shifted per member so intervals hold. Recorded as D-02.
