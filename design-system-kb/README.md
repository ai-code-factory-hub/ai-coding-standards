# Design System KB — Index & Unbreakable Rules

Purpose: a brand-neutral, re-skinnable design system any project can adopt on day one — with a complete, ready-made "Indigo" skin included as the default.

## What this KB is

A **token-driven** design system built so that changing the brand is a *data* change, not a *code* change. Every project starts from a neutral placeholder palette; swapping in your own brand (or the included Indigo skin) means replacing **one block of primitive tokens** — everything else inherits.

- Components consume **semantic/component tokens only**, never raw hex.
- A theme is a **JSON token set**; switching themes = swapping the set.
- The neutral palette here is the **placeholder default**; the **Indigo** token set (Indigo/Saffron/Slate, Inter + Noto Indic) is an **included ready skin** you can swap in wholesale.

## Files in this KB

| File | Covers |
|---|---|
| [`01-design-tokens.md`](01-design-tokens.md) | Three-tier token architecture, neutral placeholder palette, type scale, weights, spacing, radius, shadow, motion — plus the included **Indigo** default skin |
| [`02-theming-and-modes.md`](02-theming-and-modes.md) | Light/dark/follow-system, theme catalog concept (≥12), dark-mode lifting, system→tenant→user override, user-definable colors, white-label across app/PDF/email |
| [`03-components-and-a11y.md`](03-components-and-a11y.md) | Component recipes (buttons, cards, badges, inputs, avatars), tabular numerals, locale formatting, WCAG 2.2 AA rules, Do/Don't |

## The unbreakable rules

These are brand-independent. They hold for the neutral default, the Indigo skin, and any skin you author.

1. **Brand color only on primary CTAs, active states, and AI features.** One primary action per screen. [MUST]
2. **Accent color only on celebratory / warmth moments** (highlights, scores, avatars, recognition, low-literacy affordances) — **max ~5% of any screen.** [MUST]
3. **Semantic colors are fixed in meaning:** success / warning / danger / info. Never replace them with the brand color; never invent new semantic roles. [MUST]
4. **60-30-10 ratio:** ~60% neutral, ~30% surface, ~10% color. [SHOULD]
5. **Only AI features get a gradient.** Nothing else gets a gradient. [MUST]
6. **Body text contrast ≥ 4.5:1. Never compromise.** [MUST]
7. **Every screen tested in both light and dark mode before merge.** [MUST]

> These map directly onto the enterprise theming requirements in [`../standards-kb/18-theming-branding.md`](../standards-kb/18-theming-branding.md) and the accessibility requirements in [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md).

## How to re-skin (the whole point)

Re-skinning is a **primitive-tier swap only**. You never touch semantic tokens, component tokens, or component code.

1. **Replace the primitive color block** — your brand ramp (`brand/50…950`), your accent ramp, and (optionally) your neutral ramp. Keep the four fixed semantic colors unless you have an accessibility reason to tune them.
2. **Replace the primitive font tokens** — `font-sans`, `font-display`, `font-mono`, and any script-fallback fonts.
3. **Do nothing else.** Semantic tokens (`--color-primary`, `--color-success`, `--color-text`, …) already point at the primitives; component tokens already point at the semantics. Buttons, cards, badges, PDFs, and emails all re-skin automatically.

```
CHANGE THIS ────────────►  and everything downstream inherits
primitive tokens            semantic tokens          component tokens
brand/600  = #4F46E5   →    --color-primary     →    --button-primary-bg
neutral/900 = #0F172A  →    --color-text        →    --input-text
font-sans  = Inter     →    --font-body         →    --button-font
```

**Checklist before you call a skin done:**
- [ ] Brand ramp has all steps 50→950 (needed for hover/pressed/disabled + dark-mode lift).
- [ ] Dark mode lifts brand + semantics one lighter step (600→400) — never reuse light hexes on near-black.
- [ ] Contrast validated: body ≥ 4.5:1, large/UI ≥ 3:1, in **both** modes.
- [ ] At least one high-contrast (AAA-targeting) and one colorblind-safe theme available.
- [ ] Fonts include the scripts your users actually read.

## Included reference skin (Indigo)

The Indigo skin (Indigo 600 `#4F46E5` primary, Saffron `#EA580C` accent, Slate neutrals, Emerald/Amber/Red/Blue semantics, Inter + Noto Indic + JetBrains Mono) is reproduced verbatim as a swap-in-ready set in [`01-design-tokens.md`](01-design-tokens.md#included-reference-skin-indigo), with a 16-theme catalog in [`02-theming-and-modes.md`](02-theming-and-modes.md). Use it as-is, or use it as a worked example of a complete skin.
