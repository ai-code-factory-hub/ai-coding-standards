# Component Library KB — Catalog & Recipes

Purpose: a **project-agnostic component catalog** for the enterprise starter kit. This KB is the **reusable spec + copy-paste recipes** a project scaffolds real components from — it is **not** a compiled package. A new project reads these recipes and generates its own `packages/ui` components, styled entirely from the design system's **semantic tokens** (never raw hex).

> Target stack for recipes: **React + TypeScript + Tailwind**, shadcn-style (a `cn()` class-merge helper, `data-*` state hooks, Radix-style ARIA). Every color/spacing value comes from [`../design-system-kb/`](../design-system-kb/README.md).

## What this KB is (and isn't)

- **IS:** an inventory + anatomy + variants + states + tokens + a11y + a concise recipe for every primitive and pattern.
- **IS:** the single source of truth for *how a component should behave and be themed*.
- **ISN'T:** a shipped npm package, a Figma file, or design-token definitions (those live in [`../design-system-kb/`](../design-system-kb/README.md)).

## How to use

1. Pick the component you need from the inventory below and open its category file.
2. Scaffold it into your project's `packages/ui/` (or app) using the **Recipe** as the starting point.
3. Wire every color/space/radius through the **semantic tokens** — see [Token mapping](#token-mapping). Do not paste hex.
4. Implement **every state** listed for the component and satisfy its **Accessibility** section.
5. Verify against the file's `## Acceptance checklist`.

## Component inventory

| Component | Category | File |
|---|---|---|
| Button (primary / secondary / ghost / danger / AI-gradient) | Forms | [`01-forms.md`](01-forms.md) |
| Input | Forms | [`01-forms.md`](01-forms.md) |
| Textarea | Forms | [`01-forms.md`](01-forms.md) |
| Select | Forms | [`01-forms.md`](01-forms.md) |
| Combobox (async, typeahead) | Forms | [`01-forms.md`](01-forms.md) |
| Checkbox | Forms | [`01-forms.md`](01-forms.md) |
| Radio / RadioGroup | Forms | [`01-forms.md`](01-forms.md) |
| Switch | Forms | [`01-forms.md`](01-forms.md) |
| DatePicker | Forms | [`01-forms.md`](01-forms.md) |
| FormField (label + error + hint) | Forms | [`01-forms.md`](01-forms.md) |
| Table / DataGrid (sortable, paginated, tabular-nums) | Data display | [`02-data-display.md`](02-data-display.md) |
| List | Data display | [`02-data-display.md`](02-data-display.md) |
| Card | Data display | [`02-data-display.md`](02-data-display.md) |
| StatCard / KPI | Data display | [`02-data-display.md`](02-data-display.md) |
| Badge / Tag | Data display | [`02-data-display.md`](02-data-display.md) |
| Avatar | Data display | [`02-data-display.md`](02-data-display.md) |
| Tooltip | Data display | [`02-data-display.md`](02-data-display.md) |
| DescriptionList | Data display | [`02-data-display.md`](02-data-display.md) |
| Toast | Feedback & overlays | [`03-feedback-and-overlays.md`](03-feedback-and-overlays.md) |
| Alert / Banner | Feedback & overlays | [`03-feedback-and-overlays.md`](03-feedback-and-overlays.md) |
| Modal / Dialog | Feedback & overlays | [`03-feedback-and-overlays.md`](03-feedback-and-overlays.md) |
| Drawer / Sheet | Feedback & overlays | [`03-feedback-and-overlays.md`](03-feedback-and-overlays.md) |
| Skeleton | Feedback & overlays | [`03-feedback-and-overlays.md`](03-feedback-and-overlays.md) |
| EmptyState | Feedback & overlays | [`03-feedback-and-overlays.md`](03-feedback-and-overlays.md) |
| ConfirmDialog | Feedback & overlays | [`03-feedback-and-overlays.md`](03-feedback-and-overlays.md) |
| ProgressBar / Spinner | Feedback & overlays | [`03-feedback-and-overlays.md`](03-feedback-and-overlays.md) |
| Tabs | Navigation | [`04-navigation.md`](04-navigation.md) |
| Breadcrumb | Navigation | [`04-navigation.md`](04-navigation.md) |
| Pagination | Navigation | [`04-navigation.md`](04-navigation.md) |
| DropdownMenu | Navigation | [`04-navigation.md`](04-navigation.md) |
| Sidebar / Nav (position, collapse) | Navigation | [`04-navigation.md`](04-navigation.md) |
| Stepper | Navigation | [`04-navigation.md`](04-navigation.md) |
| CommandPalette / Search | Navigation | [`04-navigation.md`](04-navigation.md) |
| AppShell (nav + header + content) | Layout & patterns | [`05-layout-and-patterns.md`](05-layout-and-patterns.md) |
| FormLayout | Layout & patterns | [`05-layout-and-patterns.md`](05-layout-and-patterns.md) |
| FiltersBar | Layout & patterns | [`05-layout-and-patterns.md`](05-layout-and-patterns.md) |
| DataTablePage pattern | Layout & patterns | [`05-layout-and-patterns.md`](05-layout-and-patterns.md) |
| DetailPage pattern | Layout & patterns | [`05-layout-and-patterns.md`](05-layout-and-patterns.md) |
| Dashboard grid | Layout & patterns | [`05-layout-and-patterns.md`](05-layout-and-patterns.md) |
| **AppShell + Sidebar + Workspace switcher + Profile menu** (runtime recipe) | App shell & dashboard | [`06-app-shell-and-dashboard.md`](06-app-shell-and-dashboard.md) |
| **Theme picker (dropdown preview cards) + full-token `[data-theme]` runtime** | App shell & dashboard | [`06-app-shell-and-dashboard.md`](06-app-shell-and-dashboard.md) |
| **KPI card · AI insight card · SVG line/bar/control charts · Dashboard composition** | App shell & dashboard | [`06-app-shell-and-dashboard.md`](06-app-shell-and-dashboard.md) |

> `06-app-shell-and-dashboard.md` is the **copy-paste runtime reference** (theme dropdown, shell, charts) — start there when scaffolding the app frame, then port to React/Tailwind.

## Shared rules (apply to every component)

1. **Token-driven, never raw hex.** Every color, radius, spacing, shadow, and motion value maps to a semantic token from [`../design-system-kb/01-design-tokens.md`](../design-system-kb/01-design-tokens.md). A component that hardcodes `#4F46E5` is a bug — it won't re-skin. See [`03-components-and-a11y.md`](../design-system-kb/03-components-and-a11y.md).
2. **A11y-first (WCAG 2.2 AA).** Per [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md): keyboard-operable with logical tab order, visible focus ring on every interactive element, semantic HTML/ARIA, labelled inputs, announced errors, **never meaning by color alone** (icon + label), touch targets ≥ 44×44px, honor `prefers-reduced-motion`, 200% zoom without breakage.
3. **Every component ships all its states.** Default, hover, focus, active, disabled, loading, error, read-only — as relevant. No blank spinners; every screen has designed loading / empty / error states (see [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)).
4. **One primary CTA per screen; only AI gets a gradient.** From the design system's unbreakable rules. Status = fixed semantic color + icon + label.
5. **All strings i18n-externalized; numbers/dates/currency locale-formatted** at render (`Intl.*`), tabular numerals on numeric columns.
6. **Borders, not shadows, by default.** Radius from the scale (`rounded-lg` buttons, `rounded-md` inputs, `rounded-xl` cards).

## Token mapping

Recipes reference the design system's CSS custom properties directly through Tailwind arbitrary values so the token dependency is explicit:

| Role | CSS var | Tailwind in recipes |
|---|---|---|
| Primary | `--color-primary` | `bg-[--color-primary]`, `text-[--color-primary]`, `ring-[--color-primary]` |
| Accent | `--color-accent` | `text-[--color-accent]` |
| Success / Warning / Danger / Info | `--color-success` … | `text-[--color-danger]`, `bg-[--color-warning]` |
| Page bg / Surface / Border | `--color-bg` / `--color-surface` / `--color-border` | `bg-[--color-bg]`, `bg-[--color-surface]`, `border-[--color-border]` |
| Text / Muted | `--color-text` / `--color-text-muted` | `text-[--color-text]`, `text-[--color-text-muted]` |
| AI gradient | `--color-ai-gradient` | `bg-[image:--color-ai-gradient]` |

> A project may instead map these to shadcn-style named Tailwind tokens (`bg-primary`, `text-foreground`, `border-border`) in `tailwind.config` — the *names* differ, the *semantic mapping* must not. Either way: **no hex in component code.**

Shared conventions used throughout the recipes:

- `cn(...)` — the shadcn class-merge helper (`clsx` + `tailwind-merge`).
- Focus ring (mandatory, never removed): `focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30`.
- Motion respects reduced-motion: `motion-safe:transition-colors motion-safe:duration-150`.
- Tabular numerals on data: `tabular-nums`.

## Cross-links

- Design tokens & skin: [`../design-system-kb/README.md`](../design-system-kb/README.md), [`../design-system-kb/01-design-tokens.md`](../design-system-kb/01-design-tokens.md)
- Component recipes & a11y source of truth: [`../design-system-kb/03-components-and-a11y.md`](../design-system-kb/03-components-and-a11y.md)
- Frontend/UX standards: [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md)
- i18n & accessibility standards: [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md)
