# 15 · AI / LLM Governance & Safety

Standards for embedding AI/LLM capabilities in a multi-tenant SaaS safely — covering trust boundaries, grounding, human oversight, cost, versioning, and data protection.

## Trust boundaries

- **[MUST]** Treat all model **input and output as untrusted**. An LLM is **not an authorization boundary** — never let model output decide access, run privileged actions, or build SQL/commands unchecked.
- **[MUST]** Defend against **prompt injection**: separate system instructions from user/document content (a malicious uploaded file could carry "ignore previous instructions"). If AI calls tools, **each tool enforces its own authz + tenant scope**.

## Grounding & output validation

- **[MUST]** **Validate model output against a strict schema** before use; never execute it directly.
- **[MUST]** **Ground factual/quantitative figures in the system of record** — numbers shown to users come from validated source data, **never invented by the model**.
- **[MUST]** Mark AI output as **AI-generated, fallible, and flaggable**.

> **Example (healthcare):** a patient-facing AI narrative may only restate values that come from validated results; the model never originates a number, and a time-critical escalation is never delayed by AI phrasing.
> **Example (fintech/e-commerce):** an AI spending summary derives every total from the transaction ledger, labels itself as AI-generated, and lets the user report an inaccuracy.

## Human oversight

- **[MUST]** **Human-in-the-loop:** AI-proposed code, schema, or financial changes are **propose-only → human approval → CI/CD**.
- **[MUST]** AI-generated language in high-stakes or regulated surfaces requires **qualified-staff sign-off** before production; a time-critical escalation SLA is never gated on AI framing.

## Cost & rate control

- **[MUST]** Enforce **token/cost budgets + rate limits per tenant / user / feature**; cap context; cache deterministic results and stable prompts; match model tier to task; monitor spend.

## Versioning & evaluation

- **[MUST]** Version prompts, models, and configs **like code** (pinned, reviewed, roll-back-able).
- **[MUST]** Maintain a **golden eval set** and run regression evals (accuracy, safety, format, cost) on every change; keep PII-scrubbed prompt/response logs.

## Data protection

- **[MUST]** **Scrub sensitive personal data and secrets before sending anything to a provider**; keep retrieval **tenant-isolated** (never another tenant's data in context).
- **[MUST]** Require a provider **DPA with no-training-on-your-data** terms; respect **data residency**; disclose AI processing in the privacy policy; apply AI-log retention + erasure.

## Acceptance checklist

- Untrusted input/output handling with prompt-injection defense and tool-level authz + tenant scope.
- Schema-validated, grounded output that cites the system of record and is labelled AI-generated.
- Propose-only changes with human approval; qualified sign-off on high-stakes AI language.
- Per-tenant/user/feature cost + rate caps with caching and spend monitoring.
- Versioned prompts/models/configs plus a golden eval set and regression evals.
- Sensitive data scrubbed before any provider; tenant-isolated retrieval; DPA + no-training terms + residency + privacy disclosure.
