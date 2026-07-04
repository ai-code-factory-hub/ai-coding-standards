# 05 · Layout & Patterns

Page-level compositions built from the primitives in [01](01-forms.md)–[04](04-navigation.md). These are **patterns**, not new primitives — they wire existing components into consistent page shells. Token-driven, a11y-first — see the [shared rules](README.md#shared-rules-apply-to-every-component). Related: [`../design-system-kb/03-components-and-a11y.md`](../design-system-kb/03-components-and-a11y.md) · [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) · [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md).

Every page pattern ships designed **loading / empty / error** states and is responsive across the device matrix.

---

## AppShell

**Anatomy:** `Sidebar (nav)` + `Header (breadcrumb/title · search · actions · user menu)` + `main content region` + optional footer. The frame every authenticated page lives in.

**Composed from:** [Sidebar/Nav](04-navigation.md#sidebar--nav), [Breadcrumb](04-navigation.md#breadcrumb), [CommandPalette](04-navigation.md#commandpalette--search), [DropdownMenu](04-navigation.md#dropdownmenu), [Avatar](02-data-display.md#avatar).

**Variants:** sidebar expanded / collapsed / off-canvas (mobile) · with or without secondary top nav. **States:** nav collapsed/expanded · mobile drawer open · content loading/error.

**Tokens used:** page `--color-bg`, sidebar/header `--color-surface`, borders `--color-border`.

**Accessibility:** landmark regions — `<header>`, `<nav aria-label="Primary">`, `<main id="main">`, `<footer>`; a **skip-to-content** link targets `#main`; header actions are labelled; on mobile the sidebar is a focus-trapping [Drawer](03-feedback-and-overlays.md#drawer--sheet).

**Recipe:**
```tsx
export function AppShell({ nav, header, children }: { nav: React.ReactNode; header: React.ReactNode; children: React.ReactNode }) {
  return (
    <div className="flex min-h-dvh bg-[--color-bg] text-[--color-text]">
      <a href="#main" className="sr-only focus:not-sr-only focus:absolute focus:m-2 focus:rounded focus:bg-[--color-surface] focus:px-3 focus:py-2">Skip to content</a>
      {nav}
      <div className="flex min-w-0 flex-1 flex-col">
        <header className="flex h-14 items-center justify-between border-b border-[--color-border] bg-[--color-surface] px-4">{header}</header>
        <main id="main" tabIndex={-1} className="flex-1 overflow-auto p-4 md:p-6">{children}</main>
      </div>
    </div>
  );
}
```

---

## FormLayout

**Anatomy:** page title + description → sectioned form (grouped [FormField](01-forms.md#formfield-label--error--hint)s, optional two-column grid) → sticky action bar (cancel + submit). **Variants:** single-column · two-column · sectioned/card-grouped · wizard (with [Stepper](04-navigation.md#stepper)). **States:** editing · validating · submitting (disabled + loading) · success · server-error (summary [Alert](03-feedback-and-overlays.md#alert--banner)).

**Composed from:** [FormField](01-forms.md#formfield-label--error--hint) + all form controls, [Button](01-forms.md#button), [Alert](03-feedback-and-overlays.md#alert--banner), [Card](02-data-display.md#card).

**Tokens used:** section border `--color-border`, action bar `--color-surface`.

**Accessibility:** each section is a `<fieldset>`/`<legend>` or `<section aria-labelledby>`; a submit-time error summary is `role="alert"` and links to the first invalid field; single primary submit button; disable submit while submitting with `aria-busy`.

**Recipe:**
```tsx
export function FormLayout({ title, description, error, submitting, onSubmit, children }: {
  title: string; description?: string; error?: string; submitting?: boolean; onSubmit: (e: React.FormEvent) => void; children: React.ReactNode;
}) {
  return (
    <form onSubmit={onSubmit} className="mx-auto max-w-2xl">
      <div className="mb-4">
        <h1 className="text-xl font-semibold text-[--color-text]">{title}</h1>
        {description && <p className="mt-1 text-sm text-[--color-text-muted]">{description}</p>}
      </div>
      {error && <div className="mb-4"><Alert tone="danger" title="Couldn't save" urgent>{error}</Alert></div>}
      <div className="flex flex-col gap-5">{children}</div>
      <div className="sticky bottom-0 mt-6 flex justify-end gap-2 border-t border-[--color-border] bg-[--color-surface] py-3">
        <Button variant="secondary" type="button">Cancel</Button>
        <Button variant="primary" type="submit" loading={submitting}>Save</Button>
      </div>
    </form>
  );
}
```

---

## FiltersBar

**Anatomy:** row of filter controls (search input, [Select](01-forms.md#select)/[Combobox](01-forms.md#combobox), date range, toggles) + active-filter [Badge](02-data-display.md#badge--tag) chips + "Clear all". **Variants:** inline · collapsible (mobile → drawer) · with saved views. **States:** default · applied (chips + count) · loading (results fetching) · empty result → [EmptyState](03-feedback-and-overlays.md#emptystate) with clear-filters.

**Composed from:** form controls, [Badge](02-data-display.md#badge--tag), [Button](01-forms.md#button).

**Tokens used:** bar `--color-surface`, border `--color-border`, chips use Badge tokens.

**Accessibility:** wrap in `<search>` or `role="search"` region with a label; each control labelled; removing a chip is a labelled button ("Remove {filter}"); result count announced via `aria-live` after filtering.

**Recipe:**
```tsx
export function FiltersBar({ children, active, onClear }: { children: React.ReactNode; active: { key: string; label: string }[]; onClear: () => void }) {
  return (
    <div role="search" aria-label="Filters" className="flex flex-col gap-2 rounded-lg border border-[--color-border] bg-[--color-surface] p-3">
      <div className="flex flex-wrap items-center gap-2">{children}</div>
      {active.length > 0 && (
        <div className="flex flex-wrap items-center gap-2">
          {active.map((f) => <Badge key={f.key} onRemove={() => {/* remove */}}>{f.label}</Badge>)}
          <Button variant="ghost" onClick={onClear} className="h-7 px-2 text-xs">Clear all</Button>
        </div>
      )}
    </div>
  );
}
```

---

## DataTablePage pattern

**Anatomy:** page header (title + count + primary action) → [FiltersBar](#filtersbar) → [DataTable](02-data-display.md#table--datagrid) → footer [Pagination](04-navigation.md#pagination). The canonical list/index screen. **Variants:** with bulk-actions toolbar (on selection) · with row actions ([DropdownMenu](04-navigation.md#dropdownmenu)) · exportable. **States:** loading (skeleton rows) · empty (no data) · no-results (filtered) · error (retry) · bulk-selection active.

**Composed from:** [DataTable](02-data-display.md#table--datagrid), [FiltersBar](#filtersbar), [Pagination](04-navigation.md#pagination), [Button](01-forms.md#button), [EmptyState](03-feedback-and-overlays.md#emptystate), [DropdownMenu](04-navigation.md#dropdownmenu).

**Tokens used:** inherits from composed primitives; page `--color-bg`.

**Accessibility:** table has an accessible name; the primary "New" action is the single primary Button; bulk-selection count is announced; filter changes announce result counts.

**Recipe:**
```tsx
export function DataTablePage({ title, count, primaryAction, filters, table, pagination, state }: {
  title: string; count: number; primaryAction: React.ReactNode; filters: React.ReactNode;
  table: React.ReactNode; pagination: React.ReactNode; state: "ready" | "loading" | "empty" | "error";
}) {
  return (
    <div className="flex flex-col gap-4">
      <div className="flex items-center justify-between">
        <h1 className="text-xl font-semibold text-[--color-text]">{title} <span className="text-sm font-normal text-[--color-text-muted] tabular-nums">({count})</span></h1>
        {primaryAction}
      </div>
      {filters}
      {state === "empty"
        ? <EmptyState title="Nothing here yet" description="Create your first record to get started." action={primaryAction} />
        : state === "error"
        ? <EmptyState title="Couldn't load data" description="Please retry." action={<Button variant="secondary">Retry</Button>} />
        : <div className="rounded-lg border border-[--color-border]">{table}</div>}
      {state === "ready" && <div className="flex justify-end">{pagination}</div>}
    </div>
  );
}
```

---

## DetailPage pattern

**Anatomy:** [Breadcrumb](04-navigation.md#breadcrumb) → header (title + status [Badge](02-data-display.md#badge--tag) + actions) → [Tabs](04-navigation.md#tabs) or sections of [DescriptionList](02-data-display.md#descriptionlist)/[Card](02-data-display.md#card) → related lists. The canonical record view. **Variants:** tabbed · single-scroll · with side summary rail · with edit [Drawer](03-feedback-and-overlays.md#drawer--sheet). **States:** loading (skeleton) · loaded · not-found · error · edit-in-progress.

**Composed from:** [Breadcrumb](04-navigation.md#breadcrumb), [Badge](02-data-display.md#badge--tag), [Tabs](04-navigation.md#tabs), [DescriptionList](02-data-display.md#descriptionlist), [Card](02-data-display.md#card), [DropdownMenu](04-navigation.md#dropdownmenu), [Skeleton](03-feedback-and-overlays.md#skeleton).

**Tokens used:** inherits; header title `--color-text`, status via Badge semantic tokens.

**Accessibility:** single `<h1>` for the record; status conveyed by Badge (icon + text); breadcrumb marks current; tabbed sections follow Tabs a11y; not-found renders an [EmptyState](03-feedback-and-overlays.md#emptystate), not a blank page.

**Recipe:**
```tsx
export function DetailPage({ crumbs, title, status, actions, children, loading }: {
  crumbs: React.ReactNode; title: string; status?: React.ReactNode; actions?: React.ReactNode; children: React.ReactNode; loading?: boolean;
}) {
  if (loading) return <div className="flex flex-col gap-4"><Skeleton className="h-4 w-40" /><Skeleton className="h-8 w-64" /><Skeleton className="h-40 w-full" /></div>;
  return (
    <div className="flex flex-col gap-4">
      {crumbs}
      <div className="flex flex-wrap items-center justify-between gap-3">
        <div className="flex items-center gap-3">
          <h1 className="text-xl font-semibold text-[--color-text]">{title}</h1>
          {status}
        </div>
        {actions}
      </div>
      <div className="flex flex-col gap-4">{children}</div>
    </div>
  );
}
```

---

## Dashboard grid

**Anatomy:** responsive grid of widgets — [StatCard/KPI](02-data-display.md#statcard--kpi) row + [Card](02-data-display.md#card)-wrapped charts/lists. Each widget loads **independently** (one slow widget never blocks the page). **Variants:** fixed template · self-service (draggable/resizable) · role/tenant-scoped. **States:** per-widget loading (skeleton) / empty / error / ready; whole-grid responsive reflow.

**Composed from:** [StatCard/KPI](02-data-display.md#statcard--kpi), [Card](02-data-display.md#card), [Skeleton](03-feedback-and-overlays.md#skeleton), [EmptyState](03-feedback-and-overlays.md#emptystate). Charts follow the design system's chart-color guidance.

**Tokens used:** page `--color-bg`, widget surfaces `--color-surface`, borders `--color-border`; deltas use `--color-success`/`--color-danger`.

**Accessibility:** each widget is a `<section aria-labelledby>` with a heading; widgets are keyboard-reachable; charts have a text/table alternative; per-widget error/empty states are announced, not silent.

**Recipe:**
```tsx
export function DashboardGrid({ widgets }: { widgets: { id: string; title: string; span?: 1 | 2 | 3; state: "loading" | "ready" | "empty" | "error"; body: React.ReactNode }[] }) {
  return (
    <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
      {widgets.map((w) => (
        <section key={w.id} aria-labelledby={`w-${w.id}`}
          className={cn("rounded-xl border border-[--color-border] bg-[--color-surface] p-4", w.span === 2 && "lg:col-span-2", w.span === 3 && "lg:col-span-3")}>
          <h2 id={`w-${w.id}`} className="mb-3 text-sm font-semibold text-[--color-text]">{w.title}</h2>
          {w.state === "loading" ? <Skeleton className="h-32 w-full" />
            : w.state === "empty" ? <EmptyState title="No data" />
            : w.state === "error" ? <p role="alert" className="text-sm text-[--color-danger]">Failed to load</p>
            : w.body}
        </section>
      ))}
    </div>
  );
}
```

---

## Acceptance checklist

- [ ] **Composed from primitives:** patterns reuse the components in [01](01-forms.md)–[04](04-navigation.md); no new hardcoded UI.
- [ ] **Token-driven:** no raw hex; page/surface/border colors from semantic tokens.
- [ ] **All states per page:** loading (skeleton), empty ([EmptyState](03-feedback-and-overlays.md#emptystate)), error (retry), and ready — no blank spinners; dashboard widgets load independently.
- [ ] **Landmarks:** `<header>`/`<nav>`/`<main>`/`<footer>`; a working **skip-to-content** link; single `<h1>` per page.
- [ ] **A11y:** one primary action per screen; status via icon + text Badge; filter/selection changes announced via `aria-live`; overlays (mobile nav, edit drawer) trap and restore focus.
- [ ] **Responsive** across the device matrix; numbers use `tabular-nums`; strings/dates/currency locale-formatted.
