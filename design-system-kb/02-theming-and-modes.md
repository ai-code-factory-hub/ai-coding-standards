# Theming & Modes

Purpose: define the generic theming *mechanism* — light/dark/follow-system, a multi-theme catalog, dark-mode lifting, the system→tenant→user override chain, user-definable colors with guardrails, and white-label across channels. The Marg catalog appears as a worked example.

> Requirements reference: [`../standards-kb/18-theming-branding.md`](../standards-kb/18-theming-branding.md) (the [MUST] theming rules) and [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md) (contrast + a11y).

## Modes: light / dark / follow-system

- **[MUST]** Ship **light** and **dark** as first-class, both meeting WCAG 2.2 AA.
- **[MUST]** Offer **follow-system** (honor OS `prefers-color-scheme`) plus explicit light/dark override, persisted per user.
- **[MUST]** **No flash of wrong theme on load** — resolve the theme before first paint (inline the resolved token set / mode class in the document head; don't wait for hydration).

### Dark-mode lifting [MUST]

In dark mode, brand + semantic colors **lift to a lighter step (600 → 400)** so they pass contrast on near-black. **Never reuse light-mode hexes on dark backgrounds.** This is mechanical: every skin's ramp must have the 400 step available for exactly this reason.

## Theme catalog concept

A theme is just a **JSON token set** overriding a subset of primitives (see [`01-design-tokens.md`](01-design-tokens.md)). This makes a *catalog* of themes cheap: each theme swaps the brand/accent primitives (and, for a11y themes, the semantic primitives), and everything else inherits.

- **[MUST]** Provide **≥12 built-in themes** (each in light + dark) so tenants have real choice out of the box.
- **[MUST]** Always include **≥1 high-contrast theme (targets AAA)** and **≥1 colorblind-safe theme**.
- **[MUST]** In every theme, **semantic colors keep their fixed meaning** (success/warning/danger/info) — only the *brand/accent* hue changes, except in the a11y themes which tune semantics for accessibility.
- **[SHOULD]** Auto-derive a full accessible tonal scale (50–950) from a tenant's single brand color to generate a **"Brand" theme** from their logo — with contrast validation.

## Override chain: system → tenant → user

- **[MUST]** Themes are selectable at **system → tenant → user** — **later wins.** A tenant can **lock** branding; a user personalizes within what the tenant allows.
- **[MUST]** Choices **persist per user + sync** across devices/sessions.
- **[MUST]** A **safe fallback theme** (default light) renders if a resolved theme is missing or invalid — no undefined tokens ever reach a component.

```
system default ──► tenant branding (may lock) ──► user preference (within allowed)
        └────────────── resolved token set (before first paint) ──────────────┘
```

## User-definable colors (with guardrails)

- **[MUST]** Expose a **curated** set of user-customizable tokens (primary/brand, accent, notification/toast colors, sidebar/top-bar bg, focus-ring) — **each with a default + "reset to default."**
- **[MUST]** **Contrast auto-validation** on any custom color (≥4.5:1 body, ≥3:1 large/UI); **block or auto-adjust** unreadable combinations. Personalization can never break accessibility.
- **[MUST]** **Brand-locked vs. user-editable** token policy **per tenant** — admins decide what users may change.
- **[SHOULD]** Auto-generate a full tonal scale from a single picked color so hover/pressed/disabled states stay coherent; keep semantic *meaning* fixed even if the brand hue is themed.

## Navigation & density personalization

- **[MUST]** Menu position **left (default) / top / right / bottom**; items renamable + reorderable; show/hide by role + entitlement; collapsible (icon-only ↔ icon+label), remembered per user.
- **[SHOULD]** Density **comfortable / compact** (affects nav + tables + forms); pinned/favorite + recent items + search-in-menu; custom icons + grouping.

## White-label across channels

- **[SHOULD]** The resolved token set flows to **app UI, generated PDFs (reports, invoices), and email/notification templates** — so a tenant's branding is consistent everywhere, not just in-app.
- **[MUST]** Custom themes **validated (contrast + completeness) before save/publish**; invalid themes rejected with reasons.
- **[SHOULD]** Theme **versioning + migration** (old saved themes keep working); **live preview**; **export/import (JSON)**; reset-to-default; **audit** who changed branding and when. Optional preset gallery/marketplace.

---

## Example theme catalog (Marg default skin)

> Illustrates the mechanism above with the included **Marg** skin. **Primary (light)** is the 600-step; **Primary (dark)** is the lifted 400-step. **Semantic colors stay fixed** (emerald/amber/red/blue) in all themes except *High Contrast* and *Colorblind-Safe*, which tune them for accessibility. Accent stays Saffron across themes (a brand constant) unless noted. Any other skin builds its own catalog the same way.

| # | Theme | Character | Primary (light) | Primary (dark) | Accent | Notes |
|---|---|---|---|---|---|---|
| 1 | **Marg Indigo** ★ default | Brand default | `#4F46E5` | `#818CF8` | Saffron `#EA580C` | The system default |
| 2 | **Marg Saffron** | Brand-forward, warm | `#EA580C` | `#FB923C` | Indigo `#4F46E5` | Accent/primary swapped |
| 3 | **Clinical Teal** | Calm, medical | `#0D9488` | `#2DD4BF` | Saffron | Popular for diagnostics |
| 4 | **Medical Blue** | Trust, classic | `#2563EB` | `#60A5FA` | Saffron | Info-blue stays distinct (blue-500) |
| 5 | **Emerald Care** | Fresh, wellness | `#059669` | `#34D399` | Saffron | Success stays emerald + icon (avoid clash) |
| 6 | **Ocean Cyan** | Cool, modern | `#0891B2` | `#22D3EE` | Saffron | — |
| 7 | **Sky** | Light, airy | `#0284C7` | `#38BDF8` | Saffron | — |
| 8 | **Violet Lab** | Premium, distinctive | `#7C3AED` | `#A78BFA` | Saffron | AI gradient origin hue |
| 9 | **Royal Purple** | Rich, executive | `#9333EA` | `#C084FC` | Saffron | — |
| 10 | **Rose Clinic** | Soft, approachable | `#E11D48` | `#FB7185` | Indigo | Danger-red stays `#EF4444` + icon (kept distinct) |
| 11 | **Fuchsia** | Bold, energetic | `#C026D3` | `#E879F9` | Saffron | — |
| 12 | **Forest Green** | Grounded, natural | `#16A34A` | `#4ADE80` | Saffron | Success shifts to emerald-only + icon |
| 13 | **Graphite** | Monochrome, focus | `#334155` | `#CBD5E1` | Indigo | Minimal color; color reserved for status/AI |
| 14 | **Midnight** | Dark-first, premium | `#818CF8` on near-black | `#A5B4FC` | Saffron | Optimized as a dark default |
| 15 | **High Contrast** ♿ | Accessibility (AAA) | `#1D4ED8` on `#FFFFFF` | `#93C5FD` on `#000000` | `#B45309` | Strong 3px borders, AAA text targets |
| 16 | **Colorblind-Safe** ♿ | Okabe-Ito palette | `#0072B2` (blue) | `#56B4E9` | `#E69F00` (orange) | Semantic: success `#009E73`, danger `#D55E00`, warning `#F0E442` (dark text), info `#56B4E9`; always icon+label |

This catalog satisfies the ≥12 / high-contrast / colorblind-safe requirements above (Marg ships 16). Cross-check the [MUST] rules in [`../standards-kb/18-theming-branding.md`](../standards-kb/18-theming-branding.md).
