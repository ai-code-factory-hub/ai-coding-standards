# Threat Model — AI / LLM Feature

> Filled STRIDE model for features backed by an LLM (chat, RAG, summarization, agents/tools). Method & risk rating: [README.md](README.md). Blank copy: [threat-model-template.md](threat-model-template.md).
> **Owner:** Security Officer · **Extends:** [15 AI/LLM Governance](../standards-kb/15-ai-llm-governance.md), [04 Security](../standards-kb/04-security.md)

## Feature / asset

Any feature that sends context to an LLM and acts on its output — chat assistants, RAG over tenant documents, summarizers, and tool/agent flows. **Asset:** the data placed in the model's context, the actions its output can trigger, and user trust in the answers. **Core principle: the LLM is untrusted, non-deterministic input — it is never an authorization boundary and its output is never trusted for security decisions.**

## Data classification

Whatever enters the context window — often **Confidential/PII/Regulated**. Note that context may leave your boundary to a third-party model provider. → [09 Compliance & Privacy](../standards-kb/09-compliance-privacy.md), [15 AI/LLM Governance](../standards-kb/15-ai-llm-governance.md).

## Actors

| Actor | Trusted? | Notes |
|---|---|---|
| End user prompting the model | Semi-trusted | direct prompt-injection source |
| **Retrieved/uploaded content in context** | **Untrusted** | indirect prompt injection ("data as instructions") |
| **The LLM itself** | **Untrusted** | output is data, not a command; can hallucinate |
| Model provider (external API) | Semi-trusted | data leaves your boundary |
| Tools/functions the model can call | Trusted infra | but invoked at the model's request |

## Trust boundaries

- **B5 Provider boundary** — context leaves to the model API. **Prompt boundary** — user + retrieved content mix into one context (injection surface). **Tool boundary** — model output triggers real actions.

## Attack surface

The user prompt, all retrieved/RAG documents and uploaded files, system-prompt construction, the model's raw output, the tool-call arguments the model emits, and the request/response sent to the provider.

## STRIDE analysis

| # | STRIDE | Threat | How (attack path) | Mitigation `[MUST]`/`[SHOULD]` | Standards ref | Residual |
|---|---|---|---|---|---|---|
| S1 | Spoofing | **Direct prompt injection** | User input overrides the system prompt ("ignore previous instructions…") to change behavior or reveal the system prompt | `[MUST]` treat all model output as untrusted; enforce policy in code around the model, not in the prompt; don't rely on the system prompt as a security control; `[SHOULD]` input/output guardrails + instruction-hierarchy techniques | [15](../standards-kb/15-ai-llm-governance.md) | **Med** |
| S2 | Spoofing | **Indirect prompt injection (via docs)** | A malicious instruction hidden in an uploaded/retrieved document hijacks the model when it's pulled into context | `[MUST]` clearly delimit and label retrieved content as untrusted data; never let retrieved text authorize actions; sanitize/strip active content; `[SHOULD]` provenance tracking + per-source trust levels | [15](../standards-kb/15-ai-llm-governance.md) / [file-upload](file-upload.md) | Med |
| T1 | Tampering | **Output-driven injection downstream** | Model output containing markup/SQL/HTML is rendered or executed unsanitized (XSS, injection) | `[MUST]` treat model output like any untrusted string: output-encode before render, parameterize before any query, validate before use | [04](../standards-kb/04-security.md) / [11](../standards-kb/11-frontend-ux.md) | Low |
| R1 | Repudiation | **Disputed AI action/answer** | User/operator disputes what the model was told, said, or did | `[MUST]` audit prompts, retrieved sources, model+version, outputs, and any tool calls with correlation ids (masking sensitive data) | [20](../standards-kb/20-logging-audit-and-traceability.md) | Low |
| I1 | Info disclosure | **Data exfiltration via context** | Injection or crafted query makes the model reveal other users'/tenants' data present in shared context, RAG index, or history | `[MUST]` apply the **same per-user/tenant authorization to retrieval** as to any data access — filter documents *before* they enter context; never place data the user can't already access into the prompt; tenant-scope the vector store | [04](../standards-kb/04-security.md) / [multi-tenant](multi-tenant-data-access.md) | Med |
| I2 | Info disclosure | **PII leakage to provider** | Sensitive/regulated data sent to a third-party model that may log or train on it | `[MUST]` minimize/redact PII before sending; use providers with a no-training, no-retention data agreement; honor data-residency; `[SHOULD]` allow a self-hosted model for regulated data | [09](../standards-kb/09-compliance-privacy.md) / [15](../standards-kb/15-ai-llm-governance.md) | Med |
| I3 | Info disclosure | **System-prompt / secret leakage** | Model coaxed into revealing its system prompt, keys, or internal instructions | `[MUST]` keep no secrets/keys in the prompt; assume the system prompt is discoverable and design accordingly | [15](../standards-kb/15-ai-llm-governance.md) / [04](../standards-kb/04-security.md) | Med |
| D1 | DoS | **Token/cost exhaustion** | Huge or looping prompts, or agent loops, run up latency and provider bills | `[MUST]` cap input/output tokens, context size, tool-call depth, and per-user request/cost quotas; timeouts and loop limits on agents | [16](../standards-kb/16-finops-cost.md) / [03](../standards-kb/03-performance-scalability.md) | Med |
| E1 | Elevation | **Tool abuse — LLM as authz boundary** | Model is given tools (delete, email, pay, query) and injection makes it call them beyond the user's rights | `[MUST]` authorize every tool call **server-side against the end user's permissions** — the model requesting an action is not authorization; least-privilege tools; human-in-the-loop confirmation for high-impact/irreversible actions; validate tool arguments | [04](../standards-kb/04-security.md) / [15](../standards-kb/15-ai-llm-governance.md) | **Med** |
| I4/Integrity | Info/Integrity | **Hallucinated facts** | Model fabricates data presented as authoritative, driving wrong decisions | `[MUST]` ground answers in retrieved sources with citations; label AI output as AI-generated; require human review for consequential decisions; `[SHOULD]` confidence signals / abstain-when-unsure | [15](../standards-kb/15-ai-llm-governance.md) | Med |

## Top risks

1. **Tool abuse treating the LLM as authz (E1)** — the highest-impact agentic risk; authorize tool calls against the real user, always.
2. **Data exfiltration via context (I1)** — retrieval must respect the same permissions as direct access, or RAG becomes a leak.
3. **Prompt injection, direct & indirect (S1/S2)** — cannot be fully "prompted away"; contain it with code-side controls and untrusted-output handling.

## Open risks / decisions

| Item | Decision | Owner | Due |
|---|---|---|---|
| Provider data agreement | Confirm no-train/no-retain + residency; self-host for regulated data | | |
| High-impact tool actions | Require human confirmation; define the irreversible-action list | | |

---

## Mitigation checklist

- [ ] `[MUST]` LLM treated as untrusted; security enforced in code, not in the system prompt.
- [ ] `[MUST]` Retrieval filtered by the end user's own permissions + tenant *before* entering context.
- [ ] `[MUST]` Every tool call authorized server-side against the user; least-privilege tools; args validated.
- [ ] `[MUST]` Human confirmation for high-impact/irreversible tool actions.
- [ ] `[MUST]` Retrieved/uploaded content labeled untrusted; can never authorize actions (indirect injection).
- [ ] `[MUST]` Model output output-encoded/parameterized before render or query.
- [ ] `[MUST]` PII minimized/redacted before the provider; no-train/no-retain agreement; residency honored.
- [ ] `[MUST]` No secrets in prompts; system prompt assumed discoverable.
- [ ] `[MUST]` Token/context/tool-depth caps + per-user cost quotas.
- [ ] `[MUST]` Answers grounded + cited; AI output labeled; humans review consequential decisions.
- [ ] `[MUST]` Prompts, sources, model+version, outputs, tool calls audited → [20](../standards-kb/20-logging-audit-and-traceability.md).
