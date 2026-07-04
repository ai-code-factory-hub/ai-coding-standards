# Runbook · Third-Party Dependency Outage

> Fires when an external dependency (payment gateway, email/SMS, auth provider, object store, API) is down or degraded.

Template: [../runbook-template.md](../runbook-template.md) · Index: [../README.md](../README.md)

## Alert / Symptoms
- Spike in errors/timeouts on calls to a specific external host or SDK.
- Circuit breaker for the dependency trips to open.
- Vendor status page reports an incident; elevated 5xx/429 from the vendor.
- Retry storms and rising latency on flows that depend on the vendor.

## Impact
- **Users:** the dependent feature fails or slows (payments decline, emails don't send, logins fail); rest of the app may be fine if isolated.
- **Severity:** **SEV-1** if it blocks a revenue/critical path (payments, auth) with no fallback; **SEV-2/3** if degradable or non-critical.

## Quick checks
1. **Confirm it's them, not us:** is the failure isolated to one external host? Check the vendor status page.
2. **Error type:** timeouts (vendor slow), 5xx (vendor down), 429 (you're being rate-limited), or auth errors (expired key on your side).
3. **Circuit breaker state** for that dependency — open/half-open/closed.
4. **Blast radius:** which user flows depend on this call; is there a fallback/degraded path?
5. **Retry behavior:** are you amplifying the outage with aggressive retries (no backoff)?

## Diagnosis
1. **Isolate the dependency:** group errors by outbound host/SDK. If only one vendor's spans fail, it's external.
2. **Rule out self-inflicted:** expired/rotated API key, quota exhaustion (429), IP allowlist change, or DNS/cert issue on your egress — these look like an outage but are yours to fix.
3. **Scope of vendor incident:** partial (one region/endpoint) vs full; check status page + vendor incident channel for ETA.
4. **Cascade risk:** is the failing call synchronous in a hot path, causing thread/connection exhaustion and taking down unrelated features? That turns a vendor blip into your outage.

## Mitigation
_Safe / reversible first._
1. **Let the circuit breaker open** (or open it manually) so calls fail fast instead of piling up threads/connections — protects the rest of the app.
2. **Degrade gracefully:** disable/hide the dependent feature, or switch it to a queued "will retry later" mode so the core app stays up.
3. **Serve cached / last-known-good data** for read paths where staleness is acceptable.
4. **Back off retries** (exponential backoff + jitter, cap attempts) to stop amplifying the outage and clear any self-inflicted 429s.
5. **Queue writes for later replay** (idempotent) if the operation can be deferred — e.g. buffer outbound emails/webhooks until the vendor recovers.
6. **Fail over to a secondary provider** if one is configured (e.g. backup email/SMS gateway).
7. **Rotate/refresh the credential** if the root cause is an expired key on your side (not reversible-neutral, but the actual fix).

## Escalation
- **Notify status/comms:** post a customer-facing status update for affected features; keep it updated. Set expectations, don't over-promise vendor ETAs.
- Open a **vendor support ticket** / engage TAM; subscribe to their incident updates.
- Page the **owning team** for the integration to manage degraded-mode and replay.

## Prevention / follow-up
- Wrap every external call in a **circuit breaker + timeout + bounded retry with backoff**.
- Design **graceful degradation and cached fallbacks** for each critical dependency up front.
- Make deferred operations **idempotent** and queue-backed so replay is safe ([queue-backlog-and-dlq](./queue-backlog-and-dlq.md)).
- Add synthetic monitoring per dependency and alert on breaker-open events.
- **Ties to:** [../../standards-kb/05-reliability-resilience.md](../../standards-kb/05-reliability-resilience.md).
