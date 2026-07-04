# Cline / Roo Code Rules — Groundwork
# Copy this file's contents into `.clinerules` (Cline) or `.roorules` (Roo Code) at your project root.

This project uses Groundwork (groundwork/START-HERE.md, groundwork/TUTORIAL.md). Follow it on every task.

BEFORE WRITING CODE
1. Read the relevant groundwork/standards-kb/ domains — the NFR bar (multi-tenancy, security,
   reliability, logging/audit, API, a11y).
2. Adapt the matching spec in groundwork/feature-blueprints/ or groundwork/admin-kb/ for known features.
3. UI: use groundwork/design-system-kb/ tokens + groundwork/component-library-kb/ recipes. No hardcoded colors.
4. Follow this project's CLAUDE.md / DECISIONS.md. Log assumptions; don't re-ask settled things.

ALWAYS
- Tenant-scope everything; enforce with DB row-level security.
- Audit-log every significant action (groundwork/standards-kb/20-logging-audit-and-traceability.md).
- Validate all input server-side; treat client + model output as untrusted.
- WCAG 2.2 AA; loading/empty/error states everywhere.

ROLES & WORKFLOW
- Act as the roles in groundwork/agents/ when asked.
- Use the groundwork/PROMPT-PLAYBOOK.md 6-step loop; unit of work = one module/screen.
- Ship only after groundwork/standards-kb/99-master-release-gate.md passes.

Only folders present in groundwork/ apply. To disable a module, remove its folder (see TUTORIAL §5).
