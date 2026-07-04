# Engineering Doctrine · The Lean Code Doctrine

**Purpose.** A reusable, project-agnostic doctrine for writing the *least* code that fully solves the problem. Most engineering cost is not writing code — it is reading, testing, securing, migrating, and deleting it later. Every line is a liability that must earn its place. This sub-KB gives your teams and coding agents an explicit, gradable standard for questioning scope, preferring what already exists, and refusing speculative complexity.

This is an **original doctrine** synthesised from lean/YAGNI/"worse-is-better" principles and restated for enterprise SaaS starter kits. It is not a copy of any tool, linter, or vendor guide.

## The four questions (the whole doctrine in one screen)

Before writing or approving any change, answer these in order. The first "no" that stops the work is a win.

1. **Does this need to exist at all?** (YAGNI) — Is there a real, present requirement, or are we building for an imagined future? Delete the task before you delete the code.
2. **Does the standard library already do it?** — Reach for the language/framework stdlib before a third-party dependency, and before hand-rolled code.
3. **Does the platform already do it?** — Reach for a native platform feature (DB constraint, HTTP cache header, queue primitive, OS/cloud capability) before an application-layer reimplementation.
4. **What is the smallest change that works?** — One line before fifty. A function before a framework. A column before a service.

If you cannot justify a "yes" that survives all four questions, the leanest correct answer is usually **do less**.

## What this is (and is not)

- **It is** a bias-setting doctrine: when two correct solutions exist, choose the smaller, more boring, more deletable one.
- **It is not** an excuse to skip tests, error handling, security, accessibility, or observability. Those are *requirements*, not gold-plating. Leanness removes speculative and duplicated code — never the code that makes a feature correct and safe. See [../standards-kb/17-nfr-coverage-gaps.md](../standards-kb/17-nfr-coverage-gaps.md).
- **It is not** "code golf." Clarity beats cleverness. The smallest change that a teammate can read in one pass wins over a shorter, denser one.

## Intensity levels

Apply the doctrine at one of three intensities. Declare the level in the task/PR so reviewers calibrate.

| Level | Mindset | Use when |
|-------|---------|----------|
| **lite** | Nudge toward simplicity; flag obvious bloat, don't block. | Prototypes, spikes, throwaway scripts, early product discovery. |
| **standard** (default) | Actively prefer stdlib/native, delete speculative abstraction, justify every new dependency. | Normal feature work in a maintained codebase. |
| **strict** | Aggressively minimal; every abstraction, dependency, and config flag must be defended in review; rule-of-three enforced before any generalization. | Core/shared libraries, hot paths, security-sensitive code, code with many consumers. |

## When to apply

- **Always** at planning time — the cheapest code to delete is the code never scoped.
- **In code review** — as a dedicated pass hunting only for over-engineering (reinvented stdlib, unused flexibility, needless deps), separate from the correctness pass.
- **Before adding a dependency, a service, a config flag, or an abstraction layer.**
- **Not** as a reason to reopen shipped, working, tested code with no other change driver — refactor when you are already touching it (see [02-anti-duplication-and-refactoring.md](02-anti-duplication-and-refactoring.md)).

## Index

- [01-lean-code-doctrine.md](01-lean-code-doctrine.md) — the rules ([MUST]/[SHOULD]): stdlib-first, platform-native-first, delete speculative abstraction, minimize dependencies, smallest change that works, no premature generalization. With before/after examples and a decision checklist.
- [02-anti-duplication-and-refactoring.md](02-anti-duplication-and-refactoring.md) — DRY vs WET tradeoffs, detecting duplication, rule-of-three, extract/inline, a refactoring catalog, safe test-guarded dedupe, and complexity/duplication budgets in CI.

## Related standards

- [../standards-kb/03-performance-scalability.md](../standards-kb/03-performance-scalability.md) — leanness that is also fast (fewer layers, fewer allocations).
- [../standards-kb/08-testing-qa.md](../standards-kb/08-testing-qa.md) — tests are the safety net that makes deletion and dedupe safe.
- [../standards-kb/14-devex-documentation.md](../standards-kb/14-devex-documentation.md) — less code to document; document the *why* of what remains.
- [../standards-kb/17-nfr-coverage-gaps.md](../standards-kb/17-nfr-coverage-gaps.md) — the non-negotiables leanness must never cut.

## Acceptance checklist

- [ ] The doctrine's four questions are asked at planning time, not just in review.
- [ ] Every PR declares an intensity level (lite/standard/strict) so reviewers calibrate.
- [ ] A dedicated over-engineering review pass runs, separate from the correctness pass.
- [ ] New dependencies, services, config flags, and abstraction layers each carry a written justification.
- [ ] Leanness is never traded against tests, security, accessibility, or observability.
