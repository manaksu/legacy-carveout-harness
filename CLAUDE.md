# Working in a legacy carve-out project

This project moves functions out of a COBOL mainframe system into TypeScript using the model in `docs/HARNESS.md`. Read that file before doing any migration work, and follow it over any other default.

## Always

- Work one wave at a time, inside `waves/NN-<function>/`. Never touch code that is not reachable from that wave's seam.
- Before converting a unit, check `research-plan.md`. If an open item holds the unit, stop and say which item and who owns it.
- Use the shared helpers and adopted rules in `ledger.md` when you write TypeScript. Cite the rule ID in a one-line comment where it applies.
- Keep `units.json` current: every change to a unit's stage, rule or target is written there in the same change.
- Every TypeScript function names its source paragraphs in a one-line comment, for example `// from ELGB120 3000-FIND-DOS-SPAN`.

## When a parity diff appears

1. Find the first differing step in the diff report.
2. State the cause in one sentence.
3. Propose a ledger rule using `templates/ledger-rule.md`. Do not patch the TypeScript alone.
4. After the rule is adopted, re-convert every unit it touches and re-run parity.

## Never

- Store or print unmasked member data. If a record fails the masking check, discard it and report the record count only.
- Request or use write access to any mainframe database or file.
- Change behavior compared with COBOL without a decision in the wave's `decisions.md`.
- Mark a gate as passed. Gates G1, G2 and G3 are signed by people; you prepare the evidence.

## When you are asked to start a new wave

1. Create `waves/NN-<function>/` from the files in `templates/`.
2. Identify the seam (the single CALL boundary) and confirm it with the person who asked before discovery.
3. Run discovery and research. Produce `inventory.md`, `layers.md` and `research-plan.md`. Stop at G1 and summarize what needs a person.
