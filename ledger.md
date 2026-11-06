# Learning ledger

Shared by every wave. Only `adopted` rules feed the Translator. See `docs/HARNESS.md` §11 and `templates/ledger-rule.md`.

> The entries below come from the illustrative eligibility example. Keep the general rules (L-011 to L-013, L-015), which apply to most COBOL systems, and replace the rest with your own.

| Rule | Status | Rule | Found by | Changes behavior |
|---|---|---|---|---|
| L-011 | adopted | Decode COMP-3 money to integer cents. Never use floating point for amounts. | R-06 | no |
| L-012 | adopted | CYYMMDD with C=1 is 20YY. Term dates 9999999 and 99991231 mean open-ended and become null. | R-04 | no |
| L-013 | adopted | Any sort or comparison on alphanumeric keys from the mainframe uses `ebcdicCompare()`. | Parity diff, sample S3 | no |
| L-014 | adopted | Dependent coverage ends on the last day of the month the child turns 26. | Parity diff, sample S1 | no |
| L-015 | adopted | `PERFORM A THRU B` with more than one paragraph converts to one function with early returns. Fall-through is flagged for review. | Research, ELGB100 | no |
| L-016 | proposed | Reads `WITH UR` become read-committed on the replica. Diffs during the 834 load window are expected. | R-07 | yes, needs decision |
| L-017 | proposed | When spans overlap, the first covering span in sorted order wins. Kept for parity; ask the SME whether overlaps should exist at all. | Parity diff, sample S3 | no |

---

### L-013 · EBCDIC collation

| Field | Value |
|---|---|
| Status | adopted |
| Found by | Parity diff on sample S3 |
| Wave | 01-eligibility-example |
| Changes behavior vs COBOL | no |
| Units touched | 3 |

**Rule**
Any sort, comparison or range check on an alphanumeric key that came from the mainframe uses `ebcdicCompare()`. EBCDIC orders lowercase, then uppercase, then digits; ASCII orders digits first.

**Evidence**
ELGB120 2000-SORT-SPANS sorts by `SPAN-EFF-DT, PLAN-CD`. With plan codes `A12` and `9X1` on the same effective date, COBOL picks `A12` and v0.3 picked `9X1`, attaching the wrong network.

**Applies to future waves when**
The COBOL has `SORT`, `ORDER BY`, `SEARCH ALL`, or `IF a < b` on a PIC X field.

```ts
// EBCDIC order: space, lowercase, uppercase, digits. Covers the characters mainframe keys use.
const EBCDIC_ORDER = " abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789";
export function ebcdicCompare(a: string, b: string): number {
  for (let i = 0; i < Math.max(a.length, b.length); i++) {
    const x = EBCDIC_ORDER.indexOf(a[i] ?? " "), y = EBCDIC_ORDER.indexOf(b[i] ?? " ");
    if (x !== y) return x - y;
  }
  return 0;
}
```
