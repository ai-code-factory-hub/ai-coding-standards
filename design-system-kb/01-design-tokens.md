# Design Tokens

Purpose: define the three-tier token architecture and the full default token set — a brand-neutral placeholder palette first, then the included **Marg** skin as a swap-in-ready alternative.

## Token architecture (mandatory)

Three tiers. **Components consume semantic/component tokens only — never raw hex.**

```
primitive tokens   →   semantic tokens        →   component tokens
(raw palette)          (role-based)               (per-component)
brand/600  #……         --color-primary            --button-primary-bg
success/500 #……        --color-success            --badge-success-bg
neutral/900 #……        --color-text               --input-border
```

- **[MUST]** Themes are CSS custom properties (web) / theme objects (native), swapped at the root. A theme = a JSON token set; switching = swapping the set.
- **[MUST]** Every token has a default; a theme overrides a subset and inherits the rest — no undefined values. A safe fallback theme (default light) renders if a tenant/user theme is missing or invalid.
- **[SHOULD]** Tokens flow to **all channels** — app UI, generated PDFs (reports, invoices), and email/notification templates — so branding is consistent everywhere.
- **[MUST]** Re-skinning replaces the **primitive tier only** (see [`README.md`](README.md#how-to-re-skin-the-whole-point)). Semantic and component tiers are brand-agnostic and must not be edited to change a brand.

---

## Default palette — brand-neutral placeholder

This is the **placeholder** every new project starts from. Names are deliberately generic. Replace the hex values with your brand's when you skin; keep the *roles* and *structure* identical.

### Primitive: brand ramp (placeholder)

Neutral-labelled brand scale. `brand/500` is the reference brand hue; **`brand/600` is the default PRIMARY step** (light mode). Swap these hexes for your brand ramp.

| Step | Placeholder hex | Role |
|---|---|---|
| brand/50 | `#F5F6FF` | tint backgrounds |
| brand/100 | `#E9EBFF` | subtle fills |
| brand/200 | `#CED3FA` | borders on tint |
| brand/300 | `#A9B1F0` | disabled-on-brand |
| brand/400 | `#8089E6` | **dark-mode primary (lifted)** |
| brand/500 | `#5A63D8` | reference hue |
| **brand/600** | **`#4750C4`** | **PRIMARY (light)** ★ |
| brand/700 | `#3A41A3` | pressed |
| brand/800 | `#2E337F` | — |
| brand/900 | `#232760` | — |
| brand/950 | `#15173A` | AI insight card bg (dark) |

### Primitive: accent ramp (placeholder)

Used sparingly (rule 2). Swap for your accent brand hue.

| Step | Placeholder hex | Role |
|---|---|---|
| accent/50 | `#FFF6F0` | — |
| accent/100 | `#FFE8D9` | — |
| accent/200 | `#FFCFB0` | — |
| accent/400 | `#FB9A5E` | **dark-mode accent (lifted)** |
| **accent/600** | **`#E5722C`** | **ACCENT (light)** ★ |
| accent/700 | `#C05A20` | pressed |
| accent/950 | `#3F1A08` | — |

### Primitive: neutral ramp — "slate" (placeholder)

Carries ~60% of every screen. A cool-gray "slate" scale is the safe default; swap for a warmer or truer gray if your brand needs it.

| Step | Placeholder hex | Typical use |
|---|---|---|
| neutral/0 | `#FFFFFF` | base bg (light) |
| neutral/50 | `#F8FAFC` | page bg (light) |
| neutral/100 | `#F1F5F9` | surface (light) |
| neutral/200 | `#E2E8F0` | border (light) |
| neutral/300 | `#CBD5E1` | primary (dark, monochrome) |
| neutral/500 | `#64748B` | secondary text (≥4.5:1 on white) |
| neutral/800 | `#1E293B` | border (dark) |
| neutral/900 | `#0F172A` | body text (light) / card (dark) |
| neutral/950 | `#020617` | page bg (dark) |

### Primitive: fixed semantic colors [MUST]

These are **not** brand tokens. Their **meaning is fixed** and they persist across every skin (rule 3). Tune them only for High-Contrast / Colorblind-Safe themes. Light `★` / dark `★★` (lifted for near-black backgrounds).

| Role | Light | Dark (lifted) | Light bg / Dark bg |
|---|---|---|---|
| **Success** | `#10B981` | `#34D399` | `#ECFDF5` / `#022C22` |
| **Warning** | `#F59E0B` | `#FBBF24` | `#FFFBEB` / `#422006` |
| **Danger** | `#EF4444` | `#F87171` | `#FEF2F2` / `#450A0A` |
| **Info** | `#3B82F6` | `#60A5FA` | `#EFF6FF` / `#172554` |

> These are the widely-used, contrast-tested defaults (emerald/amber/red/blue family). Keep them even under a wildly different brand — a green brand still uses this specific green for *success*, plus an icon, to stay unambiguous.

### Semantic tokens (role-based — brand-agnostic)

Components reference these, never the primitives above.

| Token | Points at (light) | Points at (dark) |
|---|---|---|
| `--color-primary` | brand/600 | brand/400 |
| `--color-accent` | accent/600 | accent/400 |
| `--color-success` | success light | success dark |
| `--color-warning` | warning light | warning dark |
| `--color-danger` | danger light | danger dark |
| `--color-info` | info light | info dark |
| `--color-bg` | neutral/0 | neutral/950 |
| `--color-surface` | neutral/100 | neutral/900 |
| `--color-border` | neutral/200 | neutral/800 |
| `--color-text` | neutral/900 | neutral/100 |
| `--color-text-muted` | neutral/500 | neutral/400 |
| `--color-ai-gradient` | brand→purple gradient | brand→purple gradient |

---

## Typography

### Font primitives (placeholder — swap when skinning)

| Token | Placeholder | Use |
|---|---|---|
| `font-sans` | a neutral UI sans (e.g. system-ui / Inter) | Body, UI, all default text |
| `font-display` | same family, display cut | Large headings 24px+ |
| `font-mono` | a monospace (e.g. JetBrains Mono) | IDs, codes, hex, tabular data |
| `font-<script>` | per-script fallback family | Non-Latin scripts your users read (auto-fallback from `font-sans`) |

### Type scale — Major Third (1.25), 8 sizes

| Token | px | Role |
|---|---|---|
| `xs` | 12 | tiny labels, captions |
| `sm` | 14 | secondary text, dense tables |
| `base` | 16 | **primary body** |
| `lg` | 18 | emphasized body |
| `xl` | 20 | small headings |
| `2xl` | 24 | headings (display cut starts here) |
| `3xl` | 30 | page titles |
| `4xl` | 36 | hero / large display numbers |

### Weights — five only

`300` light (large display numbers) · `400` regular (body) · `500` medium (buttons/labels/table headers) · `600` semibold (titles/active nav) · `700` bold (alerts/AI badges).

- **[MUST]** **Tabular numerals** (`font-variant-numeric: tabular-nums`) on all data tables, invoices, and numeric columns so values align.
- **[MUST]** User font-size scaling (90/100/110/125%) + browser zoom to 200% without layout breakage.
- **[SHOULD]** Set a per-script minimum readable size where a script needs it (many non-Latin scripts want ~15–16px minimum).

---

## Spacing · Radius · Shadow · Motion

- **Spacing** (4px base): `4 / 8 / 12 / 16` (standard) `/ 20 / 24 / 32 / 48 / 64`. **Multiples of 4 only.**
- **Radius:** `sm 4` · `md 6` (inputs) · **`lg 8` (buttons, default)** · `xl 12` (cards) · `2xl 16` · `full` (avatars/pills).
- **Shadow:** default **none — use borders, not shadows.** `sm` hover-lift · `md` dropdowns · `lg` modals · **focus ring `0 0 0 3px rgb(<primary> / .30)` — mandatory a11y, never removed.**
- **Motion:** `fast 150ms` (hover) · `normal 200ms` (modal/dropdown) · `slow 300ms` (page) · `skeleton 1500ms`. No bouncy animation in serious/professional software. **[MUST] Honor `prefers-reduced-motion`** — drop non-essential transitions to near-zero duration.

---

## Included default skin — "Marg" tokens (from MargHR_Design_Tokens)

> This is the **included ready skin**, reproduced verbatim from `Standards/MargHR_Design_Tokens.docx`. The neutral palette above is the placeholder; **Marg is a complete example skin**. To adopt it, replace the primitive blocks above with the ones below — **nothing else changes**, because semantic and component tokens are unaffected.

### Marg primitive: Primary — Indigo

`50 #EEF2FF · 100 #E0E7FF · 200 #C7D2FE · 300 #A5B4FC · 400 #818CF8 · 500 #6366F1 ·` **`600 #4F46E5 ★ PRIMARY`** `· 700 #4338CA · 800 #3730A3 · 900 #312E81 · 950 #1E1B4B`

**Dark mode:** primary lifts to **`400 #818CF8`** (600 fails AA on near-black); `950` for AI insight card backgrounds.

### Marg primitive: Accent — Saffron/Orange

`50 #FFF7ED · 100 #FFEDD5 · 200 #FED7AA · 400 #FB923C ·` **`600 #EA580C ★ ACCENT`** `· 700 #C2410C · 950 #431407`

**Dark:** accent lifts to **`400 #FB923C`**. Use sparingly (rule 2).

### Marg primitive: Semantic — fixed meaning (identical to defaults)

| Role | Light | Dark (lifted) | Light bg / Dark bg |
|---|---|---|---|
| **Success** — Emerald | `#10B981` | `#34D399` | `#ECFDF5` / `#022C22` |
| **Warning** — Amber | `#F59E0B` | `#FBBF24` | `#FFFBEB` / `#422006` |
| **Danger** — Red | `#EF4444` | `#F87171` | `#FEF2F2` / `#450A0A` |
| **Info** — Blue | `#3B82F6` | `#60A5FA` | `#EFF6FF` / `#172554` |

### Marg primitive: Neutral — Slate (60% of every screen)

Light: bg `#FFFFFF`, page `slate-50 #F8FAFC`, surface `slate-100`, border `slate-200 #E2E8F0`, body text `slate-900 #0F172A`.
Dark: page `slate-950 #020617`, card `slate-900`, border `slate-800`, body text `slate-100 #F1F5F9`.

### Marg primitive: Fonts

| Token | Font |
|---|---|
| `font-sans` | **Inter** (body, UI; strong Indic support, tabular numerals) |
| `font-display` | Inter Display (headings 24px+) |
| `font-mono` | JetBrains Mono (IDs, barcodes, LOINC/SNOMED codes, hex) |
| `font-hindi / tamil / telugu / bengali` | Noto Sans (per script) — auto-fallback from Inter |

> Verified Marg contrast pairs: slate-900 on white 18.7:1 (AAA), slate-500 on white 4.85:1 (AA), indigo-600 on white 7.0:1 (AAA), white on accent-600 4.65:1 (AA); dark: slate-100 on slate-950 17.5:1, indigo-400 on slate-950 5.95:1.

See [`02-theming-and-modes.md`](02-theming-and-modes.md) for the 16-theme catalog built on this skin, and [`03-components-and-a11y.md`](03-components-and-a11y.md) for component recipes.
