# Components & Accessibility

Purpose: brand-neutral component recipes (all built from semantic tokens) plus the accessibility rules every skin must satisfy — WCAG 2.2 AA, never color-alone, 44×44 touch, focus rings.

> Accessibility + locale requirements reference: [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md). Theming rules: [`../standards-kb/18-theming-branding.md`](../standards-kb/18-theming-branding.md).

## Recipe principles

- Every recipe references **semantic/component tokens** (`--color-primary`, `--color-success`, `--color-border`, …) — **never raw hex** (see [`01-design-tokens.md`](01-design-tokens.md)). This is why the same recipes re-skin for free.
- Borders, not shadows, by default. Radius from the scale (`lg 8` buttons, `xl 12` cards).
- Every interactive element carries a **visible focus ring** and an accessible name.

## Buttons

| Variant | Recipe | Rule |
|---|---|---|
| **Primary** | bg `--color-primary`, text on-primary, radius `lg`, weight `500` | **One per screen** (rule 1) |
| **Secondary** | transparent bg, `1px` `--color-border`, text `--color-text` | — |
| **Ghost / tertiary** | no border, text `--color-primary`, hover surface tint | — |
| **Danger** | bg `--color-danger` — only for destructive confirm | pair with clear label |
| **AI action** | **indigo→purple gradient** (`--color-ai-gradient`) + ✨ icon | **only AI gets a gradient** (rule 5) |
| **Disabled** | `--color-primary` at ~300-step tint, no shadow, `cursor: not-allowed` | keep ≥3:1 where it conveys state |

- **[MUST]** All buttons have an `aria-label` (or visible text label). Icon-only buttons **must** carry `aria-label`.
- **[SHOULD]** A voice/mic affordance uses the accent color to aid low-literacy users — an accent (rule 2) use, kept ≤5% of screen.

## Cards

| Variant | Recipe |
|---|---|
| **Standard** | bg `--color-surface`, `1px` `--color-border`, radius `xl`, no shadow (hover `sm` lift optional) |
| **AI insight** | tinted bg from brand `950/50` step, subtle gradient edge, ✨ badge, `--color-ai-gradient` accents — the **only** gradient surface |

## Status badges [MUST icon + label]

Status **never** relies on color alone (rule 3 + a11y). Every badge = **fixed semantic color + icon + text label.**

| State | Token | Icon | Example label |
|---|---|---|---|
| Success | `--color-success` | ✓ | "Approved" / "Paid" / "Normal" |
| Warning | `--color-warning` | ! | "Pending" / "Due soon" |
| Danger | `--color-danger` | × / ▲ | "Rejected" / "Critical" |
| Info | `--color-info` | i | "FYI" / neutral notice |
| AI | `--color-ai-gradient` | ✨ | "AI insight" / "Anomaly flag" |

> **Example:** clinical reference-range flags (H / L / HH / LL) use danger/warning tokens **plus the letter flag**, satisfying "never color-alone" on screen and in print.

## Inputs

- bg `--color-bg`, `1px` `--color-border`, radius `md`, text `--color-text`, placeholder `--color-text-muted`.
- Focus: `--color-border` → `--color-primary` **plus** the mandatory focus ring `0 0 0 3px rgb(<primary>/.30)`.
- Error: border `--color-danger` + inline message with icon (not color alone).
- **[MUST]** Every input has an associated, programmatic label.

## Avatars

- Radius `full`; initials or image; **accent** color allowed as a warmth moment (rule 2). Fallback initials must keep ≥4.5:1 text contrast on the generated background.

## Numerals & locale formatting

- **[MUST]** **Tabular numerals** (`font-variant-numeric: tabular-nums`) on all data tables, invoices, and numeric columns so digits align vertically.
- **[MUST]** Format numbers, currency, and dates via a locale API (`Intl.NumberFormat` / `Intl.DateTimeFormat`) — **never hand-roll grouping.** Provide a `formatCurrency` helper wired to the active locale.
- **[SHOULD]** Round numbers before display; right-align numeric columns.

> **Example:** for `en-IN`, `Intl.NumberFormat('en-IN', …)` produces the lakh/crore grouping (1,00,000 / 1,00,00,000); a `formatINR` helper wraps it for all currency. Devanagari and similar scripts want ~15–16px minimum for readability.

## Accessibility rules [MUST]

- **[MUST]** Both light + dark meet **WCAG 2.2 AA**; the High-Contrast theme targets **AAA**. Validate body ≥ 4.5:1 and large/UI ≥ 3:1 in **both** modes.
- **[MUST]** **Never convey meaning by color alone** — status uses **icon + label** (✓ Approved / ! Pending / × Rejected), not just a colored dot. Ship a colorblind-safe theme and test with a CVD simulator (~8% of men have some color-vision deficiency).
- **[MUST]** **Touch targets ≥ 44×44px** on mobile.
- **[MUST]** **Mandatory visible focus rings**; never remove the outline without an equivalent replacement.
- **[MUST]** Keyboard navigation with **logical tab order**; all buttons have `aria-label`, all inputs labelled.
- **[MUST]** Honor `prefers-reduced-motion` — drop non-essential animation.

## Do / Don't

**Do:** brand color for the **one** primary CTA per screen · test dark mode before merge · semantic colors for their meaning only · pair every status color with an icon + label · include script fallbacks for the languages your users read · format currency/numbers via the locale API · round numbers before display · **sentence case** headings.

**Don't:** use the brand color for "info" (use the fixed info-blue) · use the accent for everyday UI · put a gradient on non-AI elements · add shadows on cards by default (use borders) · show red/green status without icons · hardcode colors (use tokens) · use weights other than 400/500/600/700 (+300 for large display) · use ALL CAPS except tiny labels (e.g. URGENT, AI) · use Title Case headings.
