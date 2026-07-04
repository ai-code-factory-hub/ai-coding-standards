# 03 · Postmortem Template

**Purpose:** convert an incident into durable improvement. A postmortem exists to make the
**system** more resilient, never to assign fault. It closes the loop opened in
[00 · Incident Response Process](00-incident-response-process.md) and typically produces work
that tightens [05 · Reliability](../standards-kb/05-reliability-resilience.md),
[07 · Observability](../standards-kb/07-observability-aiops.md), and
[12 · DevOps & CI/CD](../standards-kb/12-devops-cicd.md).

## Rules of engagement

- **[MUST]** Write a postmortem for **every SEV1 and SEV2** (and any SEV3 with a useful
  lesson), within a set window (e.g. **≤5 business days** of resolution).
- **[MUST]** Be **blameless**: describe **what** happened and **why the system allowed it**,
  never **who to blame**. People acted rationally on the information they had; if a human
  "error" broke prod, the real defect is the system that let a single mistake do that.
- **[MUST NOT]** Include names in a fault-assigning way, or soften findings to protect
  feelings — psychological safety and honesty are the whole point.
- **[SHOULD]** Assume good intent; focus on missing guardrails, unclear signals, and gaps —
  the fixable things.

---

## Postmortem: `<short incident title>`

**Status:** Draft / In review / Final
**Severity:** SEV_ · **Date:** `<UTC date>` · **Duration:** `<detect→resolve>`
**Author(s):** `<role>` · **Reviewers:** `<role(s)>` · **Related incident:** `<link>`

### 1. Summary
Two or three plain-language sentences: what broke, who was affected, how long, and how it was
resolved. Readable by someone outside the team.

### 2. Impact
Quantify from the **user's** perspective:
- Users/tenants affected (count or %), and which journeys.
- SLO/error-budget consumed (see [01 · SLOs](01-slo-and-error-budgets.md)).
- Business impact: failed transactions, revenue, missed SLAs, safety/compliance exposure.
- Duration of user-visible impact (distinct from total incident length).

### 3. Timeline (UTC)
Facts and actions only — lifted from the live incident timeline.

| Time (UTC) | Event |
|---|---|
| `hh:mm` | Detection — how it was noticed (alert / synthetic / customer) |
| `hh:mm` | Incident declared, IC assigned, severity set |
| `hh:mm` | Mitigation attempted — action + result |
| `hh:mm` | User impact ends |
| `hh:mm` | Resolved / stood down |

### 4. Root cause
The **technical** chain of causation — the "5 whys" pushed past the surface trigger to the
underlying condition. The trigger is rarely the root cause: a bad deploy is a trigger; the
**absence of a canary or health-gated rollback** that let it reach everyone is closer to the
root.

### 5. Contributing factors
Conditions that made the incident **more likely, worse, or slower to resolve** — e.g. a
missing alert delayed detection, a stale runbook slowed response, a single point of failure,
thin test coverage, an ambiguous ownership boundary. Systemic, not personal.

### 6. What went well
Genuinely — what worked and should be **preserved**: fast detection, a runbook that held up,
a clean rollback, good comms. Reinforce it, don't only list failures.

### 7. What went poorly / where we got lucky
Honest gaps: slow detection, missing runbook, unclear escalation, a near-miss where luck
(not design) prevented worse. Naming luck prevents mistaking it for resilience.

### 8. Action items
The output that matters. Each is **specific, owned, dated, and prioritised** — a
recommendation with no owner is a wish.

| # | Action | Type (prevent / detect / mitigate / process) | Owner | Priority | Due | Tracking link |
|---|---|---|---|---|---|---|
| 1 | e.g. Add health-gated canary to the deploy pipeline | prevent | `<role>` | P1 | `<date>` | `<ticket>` |
| 2 | e.g. Add a burn-rate alert on the affected SLI | detect | `<role>` | P1 | `<date>` | `<ticket>` |
| 3 | e.g. Write/refresh the failover runbook | mitigate | `<role>` | P2 | `<date>` | `<ticket>` |

---

## Action items are tracked to done

A postmortem whose action items rot in a doc has changed **nothing** — you have paid the cost
of the incident and skipped the only refund.

- **[MUST]** Every action item is filed as a **tracked ticket** in the normal backlog with an
  **owner** and **due date** — not a bullet buried in a document.
- **[MUST]** P1 items are **prioritised against feature work**, not perpetually deferred; a
  repeat incident from an un-done action item is an organisational failure.
- **[SHOULD]** Review **open postmortem action items** on a regular cadence (e.g. weekly ops
  review / [retro](../standards-kb/14-devex-documentation.md)) until closed.
- **[SHOULD]** Watch for **repeat root causes** across postmortems — the same cause twice
  means the last fix didn't address the real system gap.

## Postmortem checklist

- [ ] Written for every SEV1/SEV2 within the target window.
- [ ] Blameless: system-focused language, no fault assignment.
- [ ] Impact quantified from the user's perspective (+ SLO/budget consumed).
- [ ] UTC timeline of facts and actions.
- [ ] Root cause pushed past the trigger; contributing factors listed.
- [ ] Both what-went-well and what-went-poorly captured honestly.
- [ ] Every action item owned, dated, prioritised, and filed as a tracked ticket.
- [ ] Open action items reviewed on a cadence until done; repeat causes flagged.
