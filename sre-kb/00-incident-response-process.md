# 00 · Incident Response Process

**Purpose:** a single, rehearsed way to turn "something is wrong in production" into a
coordinated response and a durable lesson. When an incident hits, nobody should be
inventing process — they run this one. Extends
[05 · Reliability & Resilience](../standards-kb/05-reliability-resilience.md) (the SLOs and
DR runbooks being defended) and [07 · Observability & AIOps](../standards-kb/07-observability-aiops.md)
(the alerting that detects the incident).

An **incident** is any unplanned event that degrades a user-facing SLO or risks data
integrity, security, or compliance — declared or not.

## Severity levels

Severity drives *urgency and who wakes up*, not blame. Classify on **customer impact**, not
on how hard the fix looks. When in doubt, **[SHOULD]** round **up** — you can always
downgrade.

| Sev | Meaning | Response | Examples |
|---|---|---|---|
| **SEV1** | Critical: core flow down or data/security breach, most/all tenants, no workaround | Page immediately, 24×7; all-hands; exec + status-page notify | Checkout totally down; auth broken for all users; suspected data breach / PII exposure; primary DB unavailable |
| **SEV2** | Major: significant degradation or a key flow broken for some tenants; workaround painful | Page on-call; war room within minutes | p99 latency 5× SLO; one region down; a large tenant fully blocked; payments failing ~10% |
| **SEV3** | Minor: partial/degraded feature, limited scope, clear workaround | Ticket + business-hours fix; no page unless it worsens | A dashboard widget failing; a non-critical export delayed; elevated but non-breaching error rate |
| **SEV4** | Low: cosmetic or negligible impact; near-miss worth recording | Backlog; batch with normal work | Minor UI glitch; a noisy but harmless log; a caught-in-time near-miss |

- **[MUST]** Every SEV1/SEV2 is **declared explicitly** (a channel, a ticket, a named
  Incident Commander) — no "we sort of handled it in DMs".
- **[SHOULD]** Re-evaluate severity as facts change and **record every change** with a
  timestamp and reason.

> **Example (healthcare):** critical result-notification pipeline stalled so clinicians
> aren't alerted → **SEV1** (patient-safety impact) even if only one tenant, because there
> is no safe workaround.
>
> **Example (e-commerce):** promo-code service down during a sale, checkout still works at
> list price → **SEV2**: real revenue impact, but a workaround exists.

## Incident lifecycle

Every incident moves through the same stages. Do not skip; do not wait for perfect
information before acting.

1. **Detect** — an alert, synthetic check, customer report, or AIOps signal fires.
   **[MUST]** Alerts tie to an SLO and link a [runbook](runbook-template.md).
2. **Triage** — assess impact and blast radius, assign **severity**, declare the incident,
   and appoint an **Incident Commander (IC)**. Start the clock.
3. **Mitigate** — **stop the bleeding before finding root cause.** Prefer the fastest safe
   lever: roll back the last deploy, flip a feature flag, fail over, shed load, scale out.
   Mitigation ≠ resolution — restoring the user comes first.
4. **Communicate** — in parallel with mitigation, not after. Internal updates on a cadence;
   external status-page + customer comms per severity (see below).
5. **Resolve** — user-facing impact ended and confirmed (SLIs recovered, synthetics green,
   no error resurgence). Declare resolved and stand down the roles.
6. **Postmortem** — for every SEV1/SEV2 (and any SEV3 with a useful lesson), run a blameless
   [postmortem](03-postmortem-template.md) with tracked action items.

- **[MUST]** Mitigate before diagnose. Understanding *why* can wait; a bleeding user cannot.
- **[MUST]** Any mitigation that changes production (rollback, flag flip, failover, manual
  data edit) is announced in the incident channel **before** it is applied and logged
  **after** — see [12 · DevOps & CI/CD](../standards-kb/12-devops-cicd.md) for the rollback
  and flag machinery.
- **[SHOULD NOT]** Make risky data-mutating changes solo; get a second pair of eyes on the
  incident channel first.

## Roles

Small incidents may collapse roles into one person; **[MUST]** separate them for SEV1 —
one person cannot command, communicate, and type fixes at once.

- **Incident Commander (IC)** — owns the response, not the keyboard. Decides severity,
  assigns tasks, keeps the timeline, runs the war room, and calls resolution. The IC's
  job is coordination and decisions, **[SHOULD NOT]** be the one debugging.
- **Comms Lead** — owns all human communication: internal cadence updates, the status page,
  customer/exec messaging, and support enablement. Shields the Ops from stakeholders.
- **Ops / Subject-Matter Expert(s)** — the hands-on responders investigating and applying
  mitigations. Report findings to the IC; do not go silent.
- **[SHOULD]** Name a **Scribe** for SEV1 (may be the Comms Lead) to keep the timeline
  clean in the moment so it isn't reconstructed from memory later.

## Status-page & customer communication

Under-communicating erodes trust faster than the outage itself. Communicate early, say what
you know, and never speculate on cause publicly.

- **[MUST]** SEV1 → public **status-page** entry within a target window (e.g. **≤15 min** of
  declaration) and updates on a **fixed cadence** (e.g. every 30 min) even when the update is
  "still investigating, no ETA yet". SEV2 → status-page at Comms Lead's discretion by scope.
- **[MUST]** Distinguish **acknowledged / identified / monitoring / resolved** so customers
  know where you are without reading internal detail.
- **[MUST NOT]** Publish root-cause speculation, blame, or PII in external updates. State
  impact and what you're doing, not theories.
- **[SHOULD]** Post a brief **public postmortem summary** for customer-facing SEV1s once the
  internal postmortem lands — accountability, not raw internals.

> **Example (fintech):** during a payment outage the status page reads "Identified — a
> subset of card payments are failing; we have disabled the affected path and are
> monitoring recovery," updated every 30 min — no mention of the upstream vendor by name
> until confirmed and cleared for disclosure.

## Timeline discipline

The timeline is the raw material of the postmortem and the only defense against
memory-rewriting.

- **[MUST]** Timestamp (UTC) every material event as it happens: detection, declaration,
  each severity change, each mitigation attempt and its result, comms sent, and resolution.
- **[MUST]** Record **facts and actions**, not judgements ("deployed flag X at 14:02", not
  "Priya broke it").
- **[SHOULD]** Capture the **detection-to-mitigation** and **detection-to-resolution**
  durations — these feed MTTA/MTTR trends and the [postmortem](03-postmortem-template.md).

## Incident checklist

- [ ] Severity assigned on customer impact; rounded up when unsure; changes logged.
- [ ] SEV1/SEV2 explicitly declared with a named IC (and Comms Lead / Ops for SEV1).
- [ ] Mitigation attempted **before** root-cause analysis; every prod change announced + logged.
- [ ] Status page + customer comms on cadence for SEV1; no speculation or PII published.
- [ ] UTC timeline kept live: detection, mitigations, comms, resolution.
- [ ] Resolution confirmed by SLIs/synthetics before stand-down.
- [ ] Blameless [postmortem](03-postmortem-template.md) scheduled for every SEV1/SEV2.
