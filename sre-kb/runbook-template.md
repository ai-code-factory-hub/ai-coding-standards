# Runbook Template

**Purpose:** the single structure **every** runbook in [`runbooks/`](runbooks/) follows, so an
on-call engineer at 3am can navigate any alert's guide without relearning the layout. A good
runbook lets a **rested, competent-but-unfamiliar** responder act correctly under stress —
copy-pasteable commands, no tribal knowledge, no prose essays.

A runbook is the action end of an alert: [07 · Observability](../standards-kb/07-observability-aiops.md)
fires the alert, [00 · Incident Response](00-incident-response-process.md) governs the
response, and **this** tells the responder exactly what to do.

## Authoring rules

- **[MUST]** Every **paging** alert links to a runbook (see
  [02 · On-Call & Escalation](02-oncall-and-escalation.md)) — a page with no runbook is a bug.
- **[MUST]** Use these sections **in this order**; omit none (write "N/A" if truly empty).
- **[MUST]** Prefer **exact, copy-pasteable** commands/queries/dashboard links over
  description. "Check the queue" is useless at 3am; the command that shows the depth is not.
- **[SHOULD]** Keep it **scannable** — short imperative steps, expected output noted, so the
  responder knows if a step worked.
- **[SHOULD]** Review/refresh after every incident that used it or exposed a stale step; a
  wrong runbook is worse than none. **[SHOULD]** date the last-verified line.
- **[MUST NOT]** Embed secrets, credentials, or PII — link to the secret store.

---

## `<Alert / failure-mode name>`

**Last verified:** `<date>` · **Owner:** `<team/role>` · **Related SLO:** `<link>`

### Alert / Symptoms
What fires this runbook and how it presents. The exact alert name(s), plus what a human
observes (error spike, latency, failed synthetic, customer report). Enough to confirm you're
in the right runbook.

### Impact
What the user feels and how bad — which journeys/tenants degrade, and the likely **severity**
(cross-ref [00 · severity levels](00-incident-response-process.md)). Frames urgency and
whether to declare an incident now.

### Quick checks (first 60 seconds)
The fastest "is it real, and how big?" triage — dashboards to open, one-line health commands,
"is it one tenant or all?", "did a deploy just go out?". Rule scope in or out before diving.

```
# e.g. links + copy-paste commands
open <dashboard-url>
<command to show current error rate / queue depth / replica status>
```

### Diagnosis steps
Ordered steps to localise the cause — check X, then Y, then Z — each with the **command** and
the **expected vs bad** output, and what each branch implies. This is the decision tree, not a
narrative.

### Mitigation steps
How to **stop the bleeding** — the fastest safe levers first (roll back, flip a feature flag,
fail over, scale out, shed load; see [12 · DevOps & CI/CD](../standards-kb/12-devops-cicd.md)).
**[MUST]** state which actions are safe to run solo vs need a second reviewer, and any
data-loss/irreversibility risk. Mitigation restores the user; it is not the root-cause fix.

```
# e.g. the actual rollback / failover / flag command
<mitigation command>
```

### Escalation
Who/what to escalate to and **when** — the timeout before escalating, the secondary, the
dependency/vendor owner, and the trigger to declare/raise a formal incident. Ties into the
[escalation policy](02-oncall-and-escalation.md).

### Prevention / Follow-up
What reduces the chance or blast radius next time — the alert-tuning, guardrail, or fix to
propose, and a pointer to file a [postmortem](03-postmortem-template.md) action item if this
recurs. Keeps the runbook a living document, not a dead end.

---

## Runbook quality checklist

- [ ] Linked from its paging alert; named for the alert/failure mode.
- [ ] All seven sections present, in order: Alert/Symptoms → Impact → Quick checks → Diagnosis → Mitigation → Escalation → Prevention/Follow-up.
- [ ] Commands/queries/links are exact and copy-pasteable; expected output noted.
- [ ] Mitigation flags solo-safe vs needs-review and any irreversibility risk.
- [ ] No secrets/PII inline; last-verified date and owner present.
- [ ] Refreshed after the last incident that touched it.
