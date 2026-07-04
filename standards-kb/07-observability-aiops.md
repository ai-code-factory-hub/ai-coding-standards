# 07 · Observability & AIOps

How the system makes itself understandable in production — structured telemetry, actionable alerting, and a human-gated AI operations loop.

## Observability foundation

- **[MUST]** **Structured JSON logs** with a consistent field set: UTC timestamp, level, message, `tenant_id`, `user_id`, `correlation_id`, service, version, environment. The correlation id is propagated across services and background jobs. **Never log secrets or sensitive personal data in plaintext** — mask or tokenize. Logs are centralized, searchable, and retained per compliance policy.
- **[MUST]** Emit the standard signal set:
  - **RED** (Rate / Errors / Duration) per endpoint,
  - **USE** (Utilization / Saturation / Errors) per resource,
  - **business metrics** (e.g. orders placed, records processed, jobs completed, SLA adherence),
  with a **per-tenant breakdown**.
- **[MUST]** APM / distributed tracing (e.g. OpenTelemetry) spanning web, DB, cache, queue, and external integrations. Alerting is **symptom-based, actionable, deduplicated, and tied to SLOs**; synthetic monitoring exercises key user journeys.

> **Example (fintech/e-commerce):** business metrics include checkout conversion, payment success rate, and settlement latency, sliced per tenant/merchant.
> **Example (healthcare):** business metrics include throughput, turnaround-time adherence, and critical-event acknowledgment SLA.

## AI operations engine (AIOps)

- **[MUST]** A daily, **PII-scrubbed** telemetry bundle feeds an AI engine that returns **ranked, human-reviewed recommendations**: top errors and root causes, performance/cost regressions, security anomalies, and capacity trends. Recommendations are reviewed by a human before any code or infrastructure change; **auto-apply is limited to safe, reversible actions**.
- **[MUST]** Client and offline errors are captured with full context, auto-synced when connectivity returns, grouped, AI-diagnosed, and ticketed.
- **[MUST NOT]** Send raw sensitive personal data, PII, or secrets to any AI provider — scrub/mask first, and respect data residency and the provider DPA.
- **[SHOULD]** Consent-respecting usage analytics produce per-tenant adoption insights ("features you are not using") and churn-risk detection.

See [AI / LLM Governance & Safety](15-ai-llm-governance.md) for model-safety controls that also apply here.

## Acceptance checklist

- Structured, sensitive-data-masked logs with propagated correlation ids.
- RED + USE + business metrics, broken down per tenant.
- Distributed tracing (APM) across all tiers and external calls.
- SLO-based, deduplicated alerting plus synthetic journey checks.
- Daily AI review is human-gated; auto-apply only safe, reversible actions.
- Offline/client errors auto-captured, grouped, and diagnosed.
- No raw sensitive data or secrets ever sent to an AI provider.
