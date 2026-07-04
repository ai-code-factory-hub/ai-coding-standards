# SRE KB — Site Reliability Engineering & On-Call (project-agnostic)

The operational half of reliability: what a team **does** when the system is live and
something breaks — how it detects, responds to, communicates, and learns from incidents,
and how it sets and defends reliability targets over time.

Written **generically** (applies to any enterprise SaaS) with **labelled examples**
across domains (e.g. healthcare, fintech, e-commerce) so each practice is concrete
without being tied to one product.

## How this extends the Standards KB

The [Standards KB](../standards-kb/README.md) says what to **build** for reliability and
observability. This KB says how to **operate** it. The two are complements — read them
together, and do not duplicate.

| Standards KB defines (build-time) | SRE KB defines (run-time) |
|---|---|
| [05 · Reliability & Resilience](../standards-kb/05-reliability-resilience.md) — SLOs exist, HA/DR, timeouts/retries/breakers, backups | How you **respond** when an SLO is breached, **run** DR, and **defend** the error budget |
| [07 · Observability & AIOps](../standards-kb/07-observability-aiops.md) — structured logs, RED/USE metrics, tracing, SLO-based alerting | How an alert becomes a **page**, who it wakes, and the **runbook** they follow |

Rule of thumb: if it is a property the software must **have**, it belongs in Standards KB.
If it is something a **human or rotation does**, it belongs here.

## How to read the rule language (RFC 2119)

- **[MUST] / [MUST NOT]** — hard requirement; no functioning on-call program without it.
- **[SHOULD] / [SHOULD NOT]** — strong default; deviation needs a written reason.
- **[MAY]** — optional; use judgement.

## Index

| # | Doc | Owns |
|---|---|---|
| 00 | [Incident Response Process](00-incident-response-process.md) | severity levels, incident lifecycle, roles, customer comms |
| 01 | [SLOs & Error Budgets](01-slo-and-error-budgets.md) | SLI/SLO/SLA, burn-rate alerting, budget policy |
| 02 | [On-Call & Escalation](02-oncall-and-escalation.md) | rotation design, escalation policy, handoff, on-call health |
| 03 | [Postmortem Template](03-postmortem-template.md) | blameless postmortem + action-item tracking |
| — | [Runbook Template](runbook-template.md) | the standard structure every runbook follows |

## Runbooks

Concrete, per-alert operational guides live in **[`runbooks/`](runbooks/)** — one file per
alert or failure mode (e.g. database failover, queue backlog, certificate expiry). Every
runbook **[MUST]** follow the shared [Runbook Template](runbook-template.md) so an on-call
engineer can navigate any of them under pressure without relearning the layout.

> The `runbooks/` folder is populated by a sibling task. This KB provides the template and
> the process those runbooks plug into.

## Related domains

- [12 · DevOps & CI/CD](../standards-kb/12-devops-cicd.md) — progressive delivery, feature flags, and health-gated rollback are your fastest mitigations.
- [16 · FinOps & Cloud Cost](../standards-kb/16-finops-cost.md) — reliability spend (redundancy, standby capacity) is a cost trade-off; error budgets frame it.

> **Source:** generalized SRE practice (Google SRE, incident-command discipline).
> Reuse across projects; extend, don't rewrite.
