# 02 · Anti-Duplication & Refactoring

**Purpose.** How to keep a codebase lean *after* the first version ships: how to reason about DRY vs WET, detect real duplication, decide when to consolidate (rule-of-three), apply a small refactoring catalog, dedupe safely behind tests, and enforce complexity/duplication budgets in CI. Pairs with [01-lean-code-doctrine.md](01-lean-code-doctrine.md), which governs code *going in*.

## DRY vs WET — a tradeoff, not a commandment

- **[MUST]** Treat **DRY** as "*every piece of **knowledge** has one authoritative representation*" — not "no two lines of code ever look alike." Deduplicate **knowledge/decisions**, not incidental textual similarity.
- **[MUST]** Prefer a little **WET** (Write Everything Twice) over a **wrong abstraction**. Coupling two things that merely *look* alike but change for different reasons creates a shared dependency that both must fight — the abstraction becomes a liability the moment their requirements diverge.
- **[SHOULD]** Ask "do these change **together, for the same reason**?" If yes → one representation. If no → let them stay separate even if they resemble each other today.

> **Example A — false DRY (before/after).**
> `Before` — one "shared" formatter coupling two unrelated rules:
> ```
> function format(value, kind) {
>   if (kind === "invoice") return `₹${value.toFixed(2)}`;   // finance rounding
>   if (kind === "report") return `${value.toFixed(1)}%`;    // display rounding
> }
> ```
> These change for different reasons (tax rules vs UI). `After` — two small, honest functions:
> ```
> const formatInvoiceAmount = v => `₹${v.toFixed(2)}`;
> const formatReportPercent = v => `${v.toFixed(1)}%`;
> ```
> Now a change to invoice rounding cannot silently break a chart label.

## Detecting duplication

- **[SHOULD]** Run a structural clone detector (e.g. **jscpd** for JS/TS/many langs, or the platform equivalent) in CI to surface **token/AST-level** clones, not just identical text. Report duplication as a percentage and per-hotspot list.
- **[SHOULD]** Distinguish clone types: **Type-1** (identical), **Type-2** (renamed identifiers), **Type-3** (near-miss, small edits). Type-1/2 across files are the strongest consolidation candidates; scattered Type-3 often signals a missing shared concept.
- **[MAY]** Track a duplication trend over time; a rising line is an early warning even when each individual clone looks harmless.

## Rule of three

- **[MUST]** Do **not** abstract on the first duplication. Copy is fine at two occurrences. **On the third**, consolidate — by then the real axis of variation is visible and the abstraction can be shaped to it.
- **[SHOULD]** When consolidating, extract to the variation you have actually observed across the three call sites — not to every hypothetical axis (see "no premature generalization" in [01-lean-code-doctrine.md](01-lean-code-doctrine.md)).

## Refactoring catalog (small, safe moves)

- **Extract Function** — pull a named, cohesive block out of a long body. **[SHOULD]** when a comment is explaining "what," replace it with a well-named function.
- **Inline Function/Variable** — the inverse; remove an indirection that no longer earns its name. **[SHOULD]** inline a one-caller helper that only obscures the flow.
- **Replace Conditional with lookup/polymorphism** — swap a growing `if/else`/`switch` on a type tag for a map/table or dispatch. **[SHOULD]** when the same switch appears in more than one place.
- **Consolidate Duplicate Conditional Fragments** — hoist a statement that runs in *every* branch out of the branches.
- **Introduce Parameter Object** — collapse a long, repeated argument list that always travels together.
- **Decompose Conditional** — extract the test, then-clause, and else-clause into named functions so intent reads in one line.
- **Remove Dead Code / Flag** — delete unreachable branches and single-value config flags (see [01](01-lean-code-doctrine.md) rule 4). Deleting is the highest-value refactor.

> **Example B — Replace Conditional with lookup (before/after).**
> `Before`:
> ```
> function labelFor(status) {
>   if (status === "paid") return "Paid";
>   if (status === "past_due") return "Past due";
>   if (status === "canceled") return "Canceled";
>   return "Unknown";
> }
> ```
> `After` — data, not control flow; new statuses are a one-line edit:
> ```
> const LABELS = { paid: "Paid", past_due: "Past due", canceled: "Canceled" };
> const labelFor = s => LABELS[s] ?? "Unknown";
> ```

## Dedupe safely (test-guarded)

- **[MUST]** Refactor only under a **green test suite** that exercises the code you are about to move. If coverage is missing, add **characterization tests** first that pin current behavior — including current quirks — then refactor.
- **[MUST]** Keep refactoring commits **behavior-preserving and separate** from feature/behavior-change commits, so a reviewer (and a `git bisect`) can trust the boundary.
- **[SHOULD]** Take **small steps**: extract → run tests → rename → run tests. Never a giant "big-bang" consolidation across many files at once.
- **[SHOULD]** Refactor **opportunistically** — improve the area you are already changing for a feature ("campground rule"), rather than scheduling risky repo-wide sweeps. See [../standards-kb/08-testing-qa.md](../standards-kb/08-testing-qa.md).

## Complexity & duplication budgets in CI

- **[MUST]** Set and enforce budgets so debt cannot grow silently:
  - **Duplication** — fail the build above a threshold (e.g. jscpd `--threshold` on % duplicated tokens); allow-list any deliberate clone with a written reason.
  - **Complexity** — cap **cyclomatic/cognitive complexity** per function (linter rule) and cap function/file length; flag, don't necessarily block, on soft limits.
- **[SHOULD]** Report the numbers as a trend in the PR so reviewers see direction, not just pass/fail. See [../standards-kb/14-devex-documentation.md](../standards-kb/14-devex-documentation.md).
- **[MAY]** Gate only on **new/changed** code (ratcheting) so legacy hotspots don't block every PR while still preventing new debt.

## Acceptance checklist

- [ ] DRY is applied to knowledge/decisions, not incidental textual similarity; false-DRY couplings are split.
- [ ] A structural clone detector runs in CI with a duplication threshold and an allow-list for deliberate clones.
- [ ] Consolidation waits for the rule of three and targets the observed axis of variation.
- [ ] Refactors run under green (or freshly added characterization) tests, in small steps, in commits separate from behavior changes.
- [ ] Cyclomatic/cognitive complexity and length budgets are enforced (ratcheting on new code where legacy exists).
- [ ] Deletion of dead code and single-value flags is treated as a first-class refactor.
