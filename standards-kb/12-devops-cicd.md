# 12 · DevOps & CI/CD

Pipeline, environment, infrastructure, and release-delivery standards for shipping a multi-tenant SaaS safely and repeatably.

## Pipeline

- **[MUST]** An automated pipeline runs on every change: build → lint/format → unit/integration tests → **security scans (SAST / SCA / secret / image)** → artifact build → deploy.
- **[MUST NOT]** Deploy to production manually. Merges are **blocked unless green and peer-reviewed**.
- **[MUST]** Artifacts are **immutable and versioned** (build once, deploy many); pipeline secrets come from a secret store, never from source.

## Environments

- **[MUST]** Maintain **dev → staging → production**. Staging **mirrors production** (config differs only by value, not shape).
- **[MUST]** No regulated or personal data in lower environments — use anonymized or synthetic data.

> **Example (healthcare):** staging is seeded with synthetic patients and de-identified results so no real health data leaves production.
> **Example (fintech/e-commerce):** load tests run against tokenized, synthetic cardholder and order data; real PANs never reach non-prod.

## Infrastructure as Code

- **[MUST]** All infrastructure is defined as **Infrastructure as Code** (Terraform/Pulumi/etc.), peer-reviewed, and applied via the pipeline.
- **[MUST NOT]** Make manual changes in the production console. State is secured and locked.

## Progressive delivery

- **[MUST]** **Zero-downtime progressive delivery** (rolling / blue-green / canary), **health-gated with auto-rollback**.
- **[MUST]** **Feature flags** decouple deploy from release — targetable per tenant / percentage / role and killable instantly; flags are inventoried and cleaned up after rollout.

## Supply chain

- **[MUST]** Dependency locking + automated updates + **SCA that fails on critical CVEs**.
- **[MUST]** Produce an **SBOM** and **sign artifacts/images**.

## Scheduling & background work

- **[MUST]** A reliable **tenant- and timezone-aware scheduler + queue/worker platform** (retries/backoff/DLQ/idempotency + visibility) runs notifications, reports, imports, external-system pushes, retention/disposal jobs, and any recurring automated runs.

## Traceability & rollback

- **[MUST]** Releases are **traceable** (version / commit / changelog / approver), support **one-click rollback**, and run **post-deploy smoke tests**.

## Acceptance checklist

- Automated, gated pipeline with no manual production deploys; merges blocked unless green and reviewed.
- Immutable, versioned artifacts; pipeline secrets from a secret store.
- dev → staging → production with staging mirroring prod and no real/regulated data in lower environments.
- All infrastructure via peer-reviewed IaC applied through the pipeline; no manual prod console changes.
- Zero-downtime progressive delivery, health-gated with auto-rollback; feature flags decoupling deploy from release.
- SCA failing on critical CVEs; SBOM produced; artifacts/images signed.
- Reliable tenant/timezone-aware scheduler + queue/worker platform with retries/backoff/DLQ/idempotency.
- Traceable releases with one-click rollback and post-deploy smoke tests.
