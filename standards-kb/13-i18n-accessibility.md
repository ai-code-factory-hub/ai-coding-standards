# 13 · Internationalization & Accessibility

Standards ensuring a multi-tenant SaaS is correct across locales, timezones, and scripts, and usable by everyone regardless of ability.

## Internationalization

- **[MUST]** **No hardcoded strings.** Use a message-formatting standard (ICU MessageFormat) with pluralization and gender support.
- **[MUST]** Store timestamps in **UTC**; display in the **tenant/user timezone** with correct DST handling.
- **[MUST]** Represent money as **amount + ISO currency code**, locale-formatted, using **decimal math** (never floating point).
- **[MUST]** Use locale-aware number and date formats, and **UTF-8 everywhere** so regional scripts (e.g. Devanagari, Tamil, Telugu, Bengali, CJK, Arabic) render correctly in names and content.
- **[MUST]** Resolve locale at **tenant and user level with fallback**; localize notification, print, and report templates too.
- **[SHOULD]** Support RTL layouts where the target locales require it.

> **Example (healthcare):** patient-facing reports and alerts are generated in the patient's chosen regional language, with names stored and rendered in their native script.
> **Example (fintech/e-commerce):** invoices show the buyer's local currency and number format, while all amounts are stored as minor units with an ISO currency code.

## Accessibility (WCAG 2.2 AA)

- **[MUST]** **Keyboard-operable** with visible focus and logical tab order (no keyboard traps).
- **[MUST]** **Semantic HTML/ARIA** — headings, landmarks, labelled inputs; associated, announced form errors.
- **[MUST]** **AA contrast** (4.5:1 text / 3:1 large text and UI components).
- **[MUST]** **Never convey meaning by color alone** — status uses icon + label, not just red/green.
- **[MUST]** Provide alt text and accessible names; **respect reduced-motion**; support text resize / 200% zoom without breakage; no flashing content.
- **[SHOULD]** Run automated a11y checks (axe) in CI and do periodic screen-reader testing.

## Acceptance checklist

- No hardcoded strings; ICU message formatting; UTF-8 throughout.
- UTC storage with tenant/user-timezone display (DST-correct).
- Money as amount + ISO code with decimal math; locale-aware number/date formats.
- RTL supported where in scope.
- WCAG 2.2 AA: keyboard operability, visible focus, semantic markup, AA contrast, not color-alone.
- Reduced-motion respected; 200% zoom without breakage; no flashing.
- axe checks in CI plus periodic screen-reader testing.
