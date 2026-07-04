# Runbook · Elevated Error Rate

> Fires when the service error ratio (5xx / failed requests / exceptions) crosses threshold above baseline.

Template: [../runbook-template.md](../runbook-template.md) · Index: [../README.md](../README.md)

## Alert / Symptoms
- Error ratio (e.g. 5xx ÷ total) exceeds SLO burn-rate threshold for N minutes.
- Spike in exception volume in the error tracker; new or resurgent error signature.
- Elevated client-visible failures, retries, and downstream error propagation.

## Impact
- **Users:** failed requests, broken flows, error pages; degree depends on which endpoints error.
- **Severity:** scale by blast radius — **SEV-1** if a core flow is broadly failing / fast burn rate; **SEV-2/3** for a contained endpoint or slow burn.

## Quick checks
1. **What changed in the last 30–60 min?** Deploy, config/flag change, migration, infra event, traffic shift.
2. **Slice the errors:** by endpoint, status code, tenant, region, instance, and version. Concentrated or global?
3. **Read the top error signature** in the tracker — stack trace, message, first-seen timestamp.
4. **Dependency health:** are DB/cache/queue/third-party calls the ones failing?
5. **Grab a correlation/trace ID** from a failing request and follow it end to end.

## Diagnosis
1. **Deploy correlation:** does error onset line up with a release or flag flip? If yes, the change is the prime suspect — see [failed-deploy-and-rollback](./failed-deploy-and-rollback.md).
2. **Correlation-ID triage:** pick a failing request's correlation ID, trace it across services, and find the first span/service that errors — that's the origin, not necessarily where the alert fired.
3. **Classify the error:**
   - 4xx surge → client/contract/auth issue or a breaking API change.
   - 5xx from app code → new exception (null/deserialization/logic) — read the stack trace.
   - 5xx from timeouts → downstream slow/unavailable ([third-party-dependency-outage](./third-party-dependency-outage.md)).
   - Auth/permission errors → expired credential, rotated key, misconfig.
4. **Config vs code:** if no code deploy, check config/secret/flag changes and infra (cert expiry, DNS, network).
5. **Scope:** single instance = bad node (evict it); single tenant = tenant data/config; global = shared dependency or release.

## Mitigation
_Safe / reversible first._
1. **Flip the feature flag off** if the errors trace to a flagged code path.
2. **Roll back the deploy** if onset correlates with a release — fastest reversible fix ([failed-deploy-and-rollback](./failed-deploy-and-rollback.md)).
3. **Revert the config/secret change** if that's the trigger.
4. **Evict/restart the bad instance** if errors are isolated to one node.
5. **Shed load / enable degraded mode** if a dependency is the cause and graceful degradation exists ([third-party-dependency-outage](./third-party-dependency-outage.md)).
6. **Hotfix forward** only if rollback isn't possible and the fix is small and well-understood.

## Escalation
- Page the **owning service team** for the erroring service.
- Declare an incident (SEV + channel + IC) when burn rate is fast or blast radius is wide.
- Loop in **dependency owners** if the origin span is a downstream/vendor service.

## Prevention / follow-up
- Propagate correlation/trace IDs across every hop; ensure structured error logging with the ID.
- Alert on SLO burn rate, not raw counts; add per-endpoint error budgets.
- Gate deploys on error-rate canary health.
- Write a blameless post-incident review with action items.
- **Ties to:** [../../standards-kb/07-observability-aiops.md](../../standards-kb/07-observability-aiops.md).
