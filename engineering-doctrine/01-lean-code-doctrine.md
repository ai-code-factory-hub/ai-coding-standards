# 01 · The Lean Code Doctrine — Rules

**Purpose.** The gradable rules of the Lean Code Doctrine. Each is marked [MUST], [SHOULD], or [MAY]. The through-line: prefer what already exists (stdlib, then platform), write the smallest change that works, and refuse code built for an imagined future. See [README.md](README.md) for the philosophy and intensity levels.

## Rule 1 — Question whether it needs to exist (YAGNI)

- **[MUST]** Build for a **present, stated requirement**, not a predicted one. "We might need X later" is not a requirement. When later arrives, the change will be cheaper on a smaller codebase.
- **[MUST]** Prefer **deleting the task** over building a small version of it. The leanest feature is the one product agreed not to ship.
- **[SHOULD]** Timebox spikes and mark them as throwaway; do not let a spike harden into production by inertia.

## Rule 2 — Standard library before custom code

- **[MUST]** Before writing a utility, confirm the language/framework standard library does not already provide it (date math, string/collection ops, hashing, UUIDs, URL/query parsing, path handling, pooling, JSON). Hand-rolled equivalents carry bugs the stdlib already fixed.
- **[SHOULD]** Prefer a well-known, correct stdlib idiom over a "clever" local one, even if the idiom is one token longer to type.

> **Example A — reinvented stdlib (before/after).**
> `Before` — a hand-written unique + grouping pass:
> ```
> const seen = {};
> const out = [];
> for (const u of users) {
>   if (!seen[u.email]) { seen[u.email] = true; out.push(u); }
> }
> ```
> `After` — stdlib does it, correctly and readably:
> ```
> const out = [...new Map(users.map(u => [u.email, u])).values()];
> ```
> Fewer lines, no manual bookkeeping object, no subtle bug when an email is the string `"constructor"`.

## Rule 3 — Platform-native before dependencies and app code

- **[MUST]** Reach for a **native platform feature** before reimplementing it in the application tier:
  - Data integrity → DB constraints (`UNIQUE`, `CHECK`, `FK`, `NOT NULL`) before app-side validation loops.
  - Caching → HTTP `Cache-Control`/`ETag` and CDN before a bespoke cache.
  - Uniqueness/ordering → DB sequences, `ON CONFLICT`, keyset pagination before app-side dedupe/sort of full sets.
  - Scheduling/retries/queues → the platform's queue/cron primitives before a custom scheduler.
- **[SHOULD]** Let the platform enforce invariants so the app cannot violate them even under a race; app-side checks are a UX nicety, not the source of truth.

> **Example B — app-layer reinvention vs native (before/after).**
> `Before` — enforce one active membership per user in app code (racy, N+1, drifts):
> ```
> const existing = await db.membership.find({ userId, active: true });
> if (existing) throw new Error("already a member");
> await db.membership.insert({ userId, active: true });
> ```
> `After` — let the database guarantee it, atomically:
> ```
> // migration: CREATE UNIQUE INDEX one_active_membership
> //   ON membership (user_id) WHERE active;
> await db.membership.insert({ userId, active: true }); // conflict = 409, no race
> ```

## Rule 4 — Delete speculative abstraction and dead flexibility

- **[MUST]** Remove config flags, plugin points, strategy interfaces, and "extensible" hooks that have **exactly one implementation and no requested second**. Flexibility with no consumer is pure cost.
- **[MUST]** Do not add a parameter "in case someone needs it." Add it when someone does.
- **[SHOULD]** Prefer a concrete, inlined implementation until a *second real* variant appears (see rule-of-three in [02-anti-duplication-and-refactoring.md](02-anti-duplication-and-refactoring.md)).

## Rule 5 — Minimize dependencies (each new one justified)

- **[MUST]** Every new third-party dependency requires a written justification covering: what stdlib/native alternative was rejected and why, the transitive footprint it pulls in, its maintenance/security posture, and the licence. A one-function helper is almost never worth a dependency.
- **[MUST]** Prefer **subtracting** a dependency when its usage has shrunk to something the stdlib now covers.
- **[SHOULD]** Pin and audit dependencies; a lean dependency graph is a smaller attack and upgrade surface. See [../standards-kb/06-data-management.md](../standards-kb/06-data-management.md) and [../standards-kb/17-nfr-coverage-gaps.md](../standards-kb/17-nfr-coverage-gaps.md).

## Rule 6 — Smallest change that works

- **[MUST]** Solve the ticket in front of you with the **smallest diff that is correct, tested, and clear** — not the most general one. Breadth is a separate, later decision.
- **[SHOULD]** Prefer editing an existing function/file over introducing a new module, service, or layer.
- **[MAY]** Leave a `ponytail:`/`TODO:` marker for a deliberate deferral so it is tracked rather than forgotten — but only for genuine deferrals, not missing correctness.

## Rule 7 — No premature generalization

- **[MUST]** Do not generalize on the first occurrence. Two concrete, slightly-duplicated call sites are cheaper and clearer than one wrong abstraction that both must now bend around.
- **[SHOULD]** When you do generalize, generalize to the *actual* variation observed across real call sites, not to every axis you can imagine.

## Decision checklist — "do we need this at all?"

Run top to bottom; stop at the first line that ends the work.

1. **Requirement** — Is there a present, stated need? If no → **don't build it.**
2. **Stdlib** — Does the standard library already do it? If yes → **use it.**
3. **Platform** — Does a native platform feature already do it? If yes → **use it.**
4. **Existing code** — Does something in this repo already do it (or 90% of it)? If yes → **reuse/extend, don't fork.**
5. **Dependency** — If reaching for a package: is the justification (rule 5) written and accepted? If no → **stop.**
6. **Abstraction** — Is there a *second real* consumer today? If no → **inline it, don't abstract.**
7. **Size** — Is this the smallest diff that is correct, tested, and clear? If no → **cut it down.**
8. **Deletion** — Could you ship this by *removing* code instead of adding it? If yes → **prefer the deletion.**

## Acceptance checklist

- [ ] Every change traces to a present requirement, not a predicted one.
- [ ] No hand-rolled utility duplicates a stdlib function; no app-tier reimplementation of a native platform feature.
- [ ] No speculative config flags, hooks, or parameters with a single consumer and no requested second.
- [ ] Each new dependency carries a written justification; unused/shrunk dependencies are removed.
- [ ] The change is the smallest correct, tested, clear diff; generalization waited for a real second case.
- [ ] The "do we need this at all?" checklist was run and recorded in the PR.
