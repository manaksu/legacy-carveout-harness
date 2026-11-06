# Legacy Carve-out Harness

A repeatable model for moving a COBOL mainframe system into TypeScript one function at a time. Agents do the discovery, research, conversion and verification; people sign three gates. It was designed around a claims adjudication engine, with membership eligibility as the first function, but it applies to any CALL-based COBOL system.

> Everything in `waves/01-eligibility-example/` and `console/` is illustrative. Program names, tables, members and numbers are made up to show how the model is filled in. Nothing here comes from a real system.

## How it works

- **Seam:** pick one existing `CALL` boundary. A routing seam takes the same record in and out and can run the new code in shadow, canary or primary mode.
- **Units:** every COBOL paragraph behind the seam is a unit, moving through six stages from discovered to in shadow.
- **Phases:** intake, discover, research, harvest tests, convert, verify, shadow, cutover. They read left to right.
- **Gates:** G1 research sign-off, G2 parity of at least 99.90% with every diff explained, G3 cutover approval.
- **Data layers:** contract, business rules, relational data, keyed files, batch feeds, reference codes and representation. Anything the harness cannot reach becomes a research item with an owner and an action plan.
- **Learning ledger:** each research finding or parity diff becomes a rule the next wave inherits.

The full model is in [`docs/HARNESS.md`](docs/HARNESS.md).

## Repository

```
docs/HARNESS.md          the model (source of truth)
docs/runtime/            how the harness runs migrated and legacy functions after cutover
CLAUDE.md                instructions Claude Code loads automatically
ledger.md                learning ledger shared by all waves
templates/               blank files for a new wave
waves/01-eligibility-example/   a filled-in example wave
console/index.html       a console that shows a wave: model, module blow-up, layers, plan, sample run, tracker, ledger
```

## Use it in your own project

1. Copy `docs/HARNESS.md`, `CLAUDE.md`, `ledger.md` and `templates/` into your project repo.
2. Open Claude Code in that repo. It reads `CLAUDE.md` on every session, which points it at the model.
3. Ask: *"Start a new wave for &lt;function&gt;. The seam is &lt;CALLER&gt; calling &lt;PROGRAM&gt;."*
4. Claude creates `waves/01-<function>/`, runs discovery and research, and stops at G1 with a list of what needs a person.
5. Clear the ledger examples you don't need, but keep the general representation rules.

Keep golden data out of Git unless it has passed your masking check. Add `waves/*/golden/` to `.gitignore` if you are unsure.

## Console

Open `console/index.html` in a browser. It has no build step and no dependencies beyond Google Fonts. It currently reads the example data embedded in the page. A later version can load `waves/*/units.json` and `research-plan.md` directly.
