# Runbook · Failed Deploy & Rollback

> Fires when a release degrades the service — failed health checks, error/latency spike post-deploy, or a stuck rollout.

Template: [../runbook-template.md](../runbook-template.md) · Index: [../README.md](../README.md)

## Alert / Symptoms
- Post-deploy spike in error rate or latency; canary/health checks failing.
- Rollout stuck (pods crash-looping, readiness probes never pass, deployment not progressing).
- New error signature first-seen at the deploy timestamp.
- Smoke tests / synthetic checks failing right after release.

## Impact
- **Users:** broken flows or full outage depending on the change; may be limited to the canary/percentage rolled out.
- **Severity:** **SEV-1** if the release is broadly live and breaking core flows; **SEV-2** if caught at canary before full rollout.

## Quick checks
1. **Rollout state:** how far did it get — canary %, which instances/versions are serving?
2. **Health/readiness probe output** for the new version; crash-loop logs.
3. **Diff the release:** code, config, dependencies, and — critically — any DB migration in this deploy.
4. **Compare metrics old vs new version** (error rate, latency, saturation) side by side.
5. **Is a schema migration involved?** This changes rollback safety — see Diagnosis.

## Diagnosis
1. **Confirm the deploy is the cause:** onset time == deploy time, and the new version's instances show the errors while old ones are clean.
2. **Startup vs runtime failure:** crash-on-boot / failed readiness = config, missing secret, bad dependency, or migration-not-run. Runtime errors = logic/contract bug.
3. **Migration safety assessment (do this before rolling back):**
   - **Additive/backward-compatible migration** (new nullable column, new table) → old code runs fine → **safe to roll back app**.
   - **Destructive/incompatible migration** (dropped/renamed column, type change, `NOT NULL` backfill) → old code may break against the new schema → **do NOT blindly roll back**; you may need to roll forward or run a compensating migration.
4. **Config vs code:** if the artifact is fine but config/flags changed, revert the config instead of the whole release.
5. **Scope:** canary-only failure = contained; full-fleet = urgent.

## Mitigation
_Safe / reversible first._
1. **Halt the rollout** immediately (pause progressive delivery) so no more traffic shifts to the bad version.
2. **Flip the feature flag / kill-switch** off — if the risky change is behind a flag, this is the fastest, safest recovery and needs no redeploy.
3. **Roll back to the last known-good release** — provided no incompatible migration blocks it (see Diagnosis step 3). Redirect traffic to the previous version/artifact.
4. **Revert the config change** if config (not code) is the culprit.
5. **Roll forward with a hotfix** when a backward-incompatible migration makes rollback unsafe — apply a small fix or compensating migration rather than reverting the schema.
6. **Scale the old version back up** if you scaled it down during rollout, to restore capacity.

## Escalation
- Page the **release owner / on-call engineer** who shipped the change.
- Involve the **DBA** for any migration-related rollback decision — never revert schema unilaterally.
- Declare an incident with an IC if the bad version reached broad production.

## Prevention / follow-up
- **Gate deploys on health checks + canary metrics**; auto-abort/auto-rollback on canary failure ([canary](../../standards-kb/12-devops-cicd.md)).
- **Expand-then-contract migrations:** always ship schema changes backward-compatibly so app rollback stays safe.
- Put risky changes behind **feature flags** with a tested kill-switch.
- Keep the previous release warm for instant rollback.
- **Ties to:** [../../standards-kb/02-versioning-deploy-migration.md](../../standards-kb/02-versioning-deploy-migration.md) and [../../standards-kb/12-devops-cicd.md](../../standards-kb/12-devops-cicd.md).
