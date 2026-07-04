# 02 · On-Call & Escalation

**Purpose:** design an on-call program that reliably gets the right human to the right
problem fast, **without burning that human out**. On-call is the delivery mechanism for the
[incident response process](00-incident-response-process.md) and the human end of the
SLO-based alerting from [07 · Observability & AIOps](../standards-kb/07-observability-aiops.md).
A rotation that wrecks people is not sustainable reliability — it is borrowed time.

## Rotation design

- **[MUST]** A **named, scheduled rotation** covering every hour someone might need to
  respond — no "whoever notices" and no permanently-on hero. The schedule is visible to the
  whole team in advance.
- **[SHOULD]** Rotate in **weekly** shifts for most teams (long enough for context, short
  enough to recover); **[MAY]** use shorter shifts or **follow-the-sun** across time zones
  to avoid routinely paging people at 3am.
- **[SHOULD]** Keep the rotation **≥4–6 people** so each person is on-call no more than
  ~1 week in 4–6; smaller pools burn out and **[SHOULD]** be a trigger to hire or merge
  rotations, not to push harder.
- **[MUST]** Compensate or time-off-in-lieu on-call per local norms/law — unpaid,
  unacknowledged on-call breeds attrition.

## Primary / secondary

- **[MUST]** Run a **primary** (first responder) and a **secondary/backup** on every shift.
  The secondary covers when the primary doesn't ack, is unreachable, or needs a second pair
  of hands on a big incident.
- **[SHOULD]** Draw the secondary from a **different experience band** (e.g. a senior backs a
  newer primary) so escalation adds capability, not just another pager.
- **[SHOULD NOT]** Make the same person primary and secondary — the backup exists precisely
  for when the primary is the point of failure.

## Escalation policy & timeouts

Escalation is automatic and time-bound, so a missed page becomes someone else's page instead
of a silent gap.

- **[MUST]** Define an explicit escalation chain with **ack timeouts**, e.g.:
  1. Page **primary** → ack within **5 min**.
  2. No ack → auto-escalate to **secondary** → ack within **5 min**.
  3. No ack → escalate to **team lead / EM**, then **incident manager / exec** for SEV1.
- **[MUST]** Tune timeouts to severity — SEV1 escalates in minutes; a SEV3 ticket does not
  need a 3am chain.
- **[SHOULD]** Have a documented path to reach **dependency owners** (DBA, network, a vendor's
  support line, another team's on-call) — many incidents aren't fixable inside one team.
- **[MUST]** Test the escalation path routinely (a periodic test page) — an untested pager
  config is discovered broken at the worst moment.

## Paging vs ticketing

Protect the pager. Every page is an interrupt to a human's sleep or focus, so the bar is
"a human must act **now**."

- **[MUST]** **Page** only for **urgent, actionable** conditions — an SLO is burning fast
  (see [error budgets](01-slo-and-error-budgets.md)) and a person must intervene. Every page
  **[MUST]** carry a [runbook](runbook-template.md) link.
- **[MUST]** **Ticket** (no page) for anything that can wait for business hours: slow-burn
  budget alerts, SEV3/SEV4, capacity trends, non-urgent AIOps recommendations.
- **[MUST NOT]** Page on **non-actionable** alerts ("CPU 80%") or on something the responder
  can't fix — that is pure fatigue. If a page has no action, delete the alert or turn it into
  a ticket.
- **[SHOULD]** Every page should be answerable by "what do I do?" — if the runbook says
  "wait and see," it wasn't a page.

> **Example (healthcare):** the result-notification pipeline stalling **pages** (patient
> safety, act now, clear runbook). A slow rise in nightly-report duration that's still within
> budget opens a **ticket** for the next working day.

## Handoff checklist

A shift change is a mini-incident risk — context is lost exactly when a smooth transfer
matters. **[MUST]** do a structured handoff, not a "you're up" DM.

The outgoing on-call hands the incoming on-call:

- [ ] **Open/ongoing incidents** and their current state + IC.
- [ ] **Recently mitigated but not root-caused** issues that could recur this shift.
- [ ] **Silenced/snoozed alerts** and when the silence expires (so nothing stays muted unseen).
- [ ] **In-flight risky changes** — deploys, migrations, maintenance windows during the shift.
- [ ] **Known-flaky** components and any temporary workarounds in place.
- [ ] **Pending follow-ups / action items** owed from earlier incidents.

- **[SHOULD]** Do the handoff **live** (call/sync message), not just a doc dump, so the
  incoming engineer can ask questions.

## On-call health & sustainable load

Alert fatigue is a reliability risk: a responder buried in noise misses the real page. Treat
on-call load as a metric you manage, not a cost you ignore.

- **[MUST]** Track **alert volume per shift** and **pages during sleep hours**. A widely-used
  ceiling: **≤2 truly actionable pages per on-call shift** on average — beyond that, fix the
  alerts, don't ask people to endure them.
- **[MUST]** Review **every** page for "was this actionable and necessary?" Non-actionable
  pages are **bugs in the alerting**; delete, re-threshold, or downgrade to ticket.
- **[SHOULD]** Budget explicit time for **operational work** (runbook fixes, alert tuning,
  toil reduction) — treat toil as debt with a cap; when it exceeds the cap, project work
  yields to reducing it.
- **[SHOULD]** After a rough shift (major incident, sleep loss), offer **recovery time** and
  fold lessons into [postmortems](03-postmortem-template.md).
- **[SHOULD NOT]** Let a single person absorb a broken rotation quietly — chronic overload is
  an org problem to fix at the schedule/staffing level, surfaced in retros.

## On-call checklist

- [ ] Named, scheduled rotation with primary + distinct secondary, published ahead.
- [ ] Escalation chain with severity-tuned ack timeouts; path to dependency/vendor owners.
- [ ] Escalation + paging config tested periodically.
- [ ] Pages are urgent + actionable + runbook-linked; everything else is a ticket.
- [ ] Structured live handoff at every shift change.
- [ ] Alert volume and sleep-hour pages tracked; every page reviewed for actionability.
- [ ] Toil/operational time budgeted; overloaded rotations escalate to staffing, not stoicism.
