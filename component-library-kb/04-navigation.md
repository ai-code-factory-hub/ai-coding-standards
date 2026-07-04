# 04 · Navigation

Wayfinding and command surfaces. Token-driven, a11y-first — see the [shared rules](README.md#shared-rules-apply-to-every-component). Related: [`../design-system-kb/03-components-and-a11y.md`](../design-system-kb/03-components-and-a11y.md) · [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) · [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md).

Navigation is landmark-based (`<nav aria-label>`), keyboard-operable, and marks the current location with `aria-current`.

---

## Tabs

**Anatomy:** tablist (tab triggers) + tab panels. **Variants:** underline · pill · vertical · with counts/badges. **States:** tab: default/hover/focus/active(selected)/disabled; panel: shown/hidden; lazy-loaded panel loading.

**Tokens used:** active indicator/text `--color-primary`, inactive `--color-text-muted`, border `--color-border`.

**Accessibility:** `role="tablist"` / `role="tab"` (`aria-selected`, `aria-controls`) / `role="tabpanel"` (`aria-labelledby`); roving tabindex; ←/→ move, Home/End jump, selected tab tabbable; automatic vs. manual activation chosen deliberately.

**Recipe:**
```tsx
import * as Tabs from "@radix-ui/react-tabs";

export function TabsView({ tabs, defaultValue }: { tabs: { value: string; label: string; content: React.ReactNode }[]; defaultValue: string }) {
  return (
    <Tabs.Root defaultValue={defaultValue}>
      <Tabs.List className="flex gap-1 border-b border-[--color-border]" aria-label="Sections">
        {tabs.map((t) => (
          <Tabs.Trigger key={t.value} value={t.value}
            className={cn("-mb-px border-b-2 border-transparent px-3 py-2 text-sm text-[--color-text-muted]",
              "data-[state=active]:border-[--color-primary] data-[state=active]:text-[--color-primary] data-[state=active]:font-medium",
              "focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30 disabled:opacity-50")}>
            {t.label}
          </Tabs.Trigger>
        ))}
      </Tabs.List>
      {tabs.map((t) => <Tabs.Content key={t.value} value={t.value} className="pt-4 focus-visible:outline-none">{t.content}</Tabs.Content>)}
    </Tabs.Root>
  );
}
```

---

## Breadcrumb

**Anatomy:** ordered trail of ancestor links → current page (not a link). Optional overflow collapse (…). **Variants:** full · collapsed (middle segments in a menu). **States:** link default/hover/focus; current (non-interactive).

**Tokens used:** links `--color-text-muted`, current `--color-text`, separator `--color-text-muted`.

**Accessibility:** `<nav aria-label="Breadcrumb">` → `<ol>`; current has `aria-current="page"`; separators are decorative (`aria-hidden`).

**Recipe:**
```tsx
export function Breadcrumb({ items }: { items: { label: string; href?: string }[] }) {
  return (
    <nav aria-label="Breadcrumb">
      <ol className="flex flex-wrap items-center gap-1 text-sm">
        {items.map((it, i) => {
          const last = i === items.length - 1;
          return (
            <li key={i} className="flex items-center gap-1">
              {it.href && !last
                ? <a href={it.href} className="text-[--color-text-muted] hover:text-[--color-text] focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30 rounded">{it.label}</a>
                : <span aria-current={last ? "page" : undefined} className="text-[--color-text]">{it.label}</span>}
              {!last && <span aria-hidden className="text-[--color-text-muted]">/</span>}
            </li>
          );
        })}
      </ol>
    </nav>
  );
}
```

---

## Pagination

**Anatomy:** prev · page numbers (with ellipsis) · next; optional page-size select + "X–Y of N". **Variants:** numbered · prev/next only · load-more · with page-size. **States:** page default/hover/focus/current/disabled (first/last).

**Tokens used:** current page `--color-primary` tint, text `--color-text`, disabled muted; counts `tabular-nums`.

**Accessibility:** `<nav aria-label="Pagination">`; current page `aria-current="page"`; prev/next labelled and `disabled` at bounds; page-size select labelled.

**Recipe:**
```tsx
export function Pagination({ page, pageCount, onPage }: { page: number; pageCount: number; onPage: (p: number) => void }) {
  const pages = Array.from({ length: pageCount }, (_, i) => i + 1); // collapse with ellipsis for large counts
  return (
    <nav aria-label="Pagination" className="flex items-center gap-1">
      <button onClick={() => onPage(page - 1)} disabled={page <= 1} aria-label="Previous page"
        className="rounded-md px-2 py-1 text-sm text-[--color-text] disabled:opacity-40 hover:bg-[--color-surface]">‹</button>
      {pages.map((p) => (
        <button key={p} onClick={() => onPage(p)} aria-current={p === page ? "page" : undefined}
          className={cn("min-w-9 rounded-md px-2 py-1 text-sm tabular-nums focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30",
            p === page ? "bg-[--color-primary]/10 text-[--color-primary] font-medium" : "text-[--color-text] hover:bg-[--color-surface]")}>{p}</button>
      ))}
      <button onClick={() => onPage(page + 1)} disabled={page >= pageCount} aria-label="Next page"
        className="rounded-md px-2 py-1 text-sm text-[--color-text] disabled:opacity-40 hover:bg-[--color-surface]">›</button>
    </nav>
  );
}
```

---

## DropdownMenu

**Anatomy:** trigger → menu popover of items (labels, icons, shortcuts, separators, submenus, checkbox/radio items). **Variants:** actions menu · with submenus · with checkbox/radio groups. **States:** trigger open/closed; item default/highlighted/disabled/checked; destructive item (danger).

**Tokens used:** menu `--color-surface` + `--color-border` + shadow `md`; highlighted item `--color-surface`; destructive `--color-danger`.

**Accessibility:** `role="menu"` / `role="menuitem"`; opens on click/Enter/Space/↓; ↑/↓ navigate, →/← for submenus, Esc closes and restores focus to trigger; type-ahead; menu not for primary navigation links (use `<nav>`).

**Recipe:**
```tsx
import * as M from "@radix-ui/react-dropdown-menu";

export function DropdownMenu({ trigger, items }: { trigger: React.ReactNode; items: { label: string; onSelect: () => void; danger?: boolean }[] }) {
  return (
    <M.Root>
      <M.Trigger asChild>{trigger}</M.Trigger>
      <M.Portal>
        <M.Content sideOffset={4} className="min-w-40 rounded-md border border-[--color-border] bg-[--color-surface] p-1 shadow-md">
          {items.map((it) => (
            <M.Item key={it.label} onSelect={it.onSelect}
              className={cn("cursor-pointer rounded px-2 py-1.5 text-sm outline-none data-[highlighted]:bg-[--color-surface]",
                it.danger ? "text-[--color-danger]" : "text-[--color-text]")}>
              {it.label}
            </M.Item>
          ))}
        </M.Content>
      </M.Portal>
    </M.Root>
  );
}
```

---

## Sidebar / Nav

**Anatomy:** brand → primary nav items (icon + label, groups, active marker) → optional footer (user/settings). Collapsible to icon-rail. **Variants:** expanded · collapsed (icon-only + tooltips) · position left/right · mobile off-canvas drawer. **States:** item default/hover/focus/active/disabled; collapsed vs expanded; group expanded/collapsed.

**Tokens used:** active item `--color-primary` + tint bg, inactive `--color-text-muted`, border `--color-border`, surface `--color-surface`.

**Accessibility:** `<nav aria-label="Primary">`; active item `aria-current="page"`; collapse toggle is a labelled button with `aria-expanded`; collapsed icon items get accessible names (visible label hidden but present, or `aria-label` + tooltip); off-canvas on mobile behaves as a [Drawer](03-feedback-and-overlays.md#drawer--sheet) (focus trap).

**Recipe:**
```tsx
export function Sidebar({ items, collapsed, currentPath }: {
  items: { href: string; label: string; icon: React.ReactNode }[]; collapsed?: boolean; currentPath: string;
}) {
  return (
    <nav aria-label="Primary" className={cn("flex h-full flex-col border-r border-[--color-border] bg-[--color-surface] p-2", collapsed ? "w-16" : "w-60")}>
      <ul className="flex flex-col gap-1">
        {items.map((it) => {
          const active = it.href === currentPath;
          return (
            <li key={it.href}>
              <a href={it.href} aria-current={active ? "page" : undefined} title={collapsed ? it.label : undefined}
                className={cn("flex items-center gap-3 rounded-lg px-3 py-2 text-sm focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30",
                  active ? "bg-[--color-primary]/10 text-[--color-primary] font-medium" : "text-[--color-text-muted] hover:bg-[--color-bg] hover:text-[--color-text]")}>
                <span aria-hidden>{it.icon}</span>
                <span className={cn(collapsed && "sr-only")}>{it.label}</span>
              </a>
            </li>
          );
        })}
      </ul>
    </nav>
  );
}
```

---

## Stepper

**Anatomy:** ordered steps (index/check · label · optional description) + connectors; used for wizards/onboarding. **Variants:** horizontal · vertical · numbered · with-icons. **States:** step complete · current · upcoming · error · disabled.

**Tokens used:** complete/current `--color-primary`, upcoming `--color-text-muted`, error `--color-danger`, connector `--color-border`. Never color-alone — completed shows a ✓, current is marked in text.

**Accessibility:** `<ol>` with each step's status in text; current step `aria-current="step"`; if steps are navigable they are buttons; error steps announce the problem.

**Recipe:**
```tsx
export function Stepper({ steps, current }: { steps: { label: string; error?: boolean }[]; current: number }) {
  return (
    <ol className="flex items-center gap-2">
      {steps.map((s, i) => {
        const done = i < current, active = i === current;
        return (
          <li key={i} aria-current={active ? "step" : undefined} className="flex items-center gap-2">
            <span className={cn("grid size-6 place-items-center rounded-full text-xs font-medium",
              s.error ? "bg-[--color-danger] text-white" : done ? "bg-[--color-primary] text-white" : active ? "ring-2 ring-[--color-primary] text-[--color-primary]" : "bg-[--color-surface] text-[--color-text-muted]")}>
              {s.error ? "!" : done ? "✓" : i + 1}
            </span>
            <span className={cn("text-sm", active ? "font-medium text-[--color-text]" : "text-[--color-text-muted]")}>{s.label}</span>
            {i < steps.length - 1 && <span aria-hidden className="h-px w-8 bg-[--color-border]" />}
          </li>
        );
      })}
    </ol>
  );
}
```

---

## CommandPalette / Search

**Anatomy:** modal overlay → search input → grouped results (recent, pages, actions) with keyboard hints → empty/loading. Opens via ⌘K/Ctrl-K. **Variants:** global command palette · scoped search · with recent/suggestions. **States:** open/closed · typing · loading · results · empty · no-permission results filtered out.

**Tokens used:** Modal tokens; highlighted result `--color-surface`; kbd hints `--color-text-muted` + `font-mono`.

**Accessibility:** dialog semantics (focus trap, Esc close, restore focus); input is `role="combobox" aria-expanded aria-controls aria-activedescendant`; result list `role="listbox"`/`option`; result count announced via `aria-live`; ↑/↓ navigate, Enter run.

**Recipe:**
```tsx
// Built on the Dialog primitive; cmdk or Radix Dialog + roving listbox.
export function CommandPalette({ open, onOpenChange, query, onQuery, results, onRun }: {
  open: boolean; onOpenChange: (o: boolean) => void; query: string; onQuery: (q: string) => void;
  results: { id: string; label: string; hint?: string }[]; onRun: (id: string) => void;
}) {
  const [active, setActive] = React.useState(0);
  return (
    <Modal open={open} onOpenChange={onOpenChange} title="Command">
      <input role="combobox" aria-expanded aria-controls="cmd-list" aria-activedescendant={`cmd-${active}`} autoFocus
        value={query} onChange={(e) => onQuery(e.target.value)} placeholder="Search commands…"
        onKeyDown={(e) => {
          if (e.key === "ArrowDown") setActive((a) => Math.min(a + 1, results.length - 1));
          if (e.key === "ArrowUp") setActive((a) => Math.max(a - 1, 0));
          if (e.key === "Enter" && results[active]) onRun(results[active].id);
        }}
        className="mb-2 h-10 w-full rounded-md border border-[--color-border] bg-[--color-bg] px-3 text-sm text-[--color-text] focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30" />
      <ul id="cmd-list" role="listbox" aria-live="polite" className="max-h-72 overflow-auto">
        {results.length === 0 && <li className="px-2 py-6 text-center text-sm text-[--color-text-muted]">No results</li>}
        {results.map((r, i) => (
          <li key={r.id} id={`cmd-${i}`} role="option" aria-selected={i === active}
            onMouseEnter={() => setActive(i)} onClick={() => onRun(r.id)}
            className={cn("flex cursor-pointer items-center justify-between rounded px-2 py-2 text-sm text-[--color-text]", i === active && "bg-[--color-surface]")}>
            {r.label}{r.hint && <kbd className="font-mono text-xs text-[--color-text-muted]">{r.hint}</kbd>}
          </li>
        ))}
      </ul>
    </Modal>
  );
}
```

---

## Acceptance checklist

- [ ] **Token-driven:** no raw hex; active/hover/current states use semantic tokens.
- [ ] **All states:** default/hover/focus/active/current/disabled + loading/empty for search surfaces; collapsed vs. expanded for sidebar.
- [ ] **Landmarks & current:** each nav region is a labelled `<nav>`; current location marked with `aria-current` (`page`/`step`).
- [ ] **Keyboard:** tabs (←/→/Home/End), menus (↑/↓/→/←/Esc/type-ahead), palette (↑/↓/Enter/Esc), pagination controls disabled at bounds; visible focus rings throughout.
- [ ] **Overlays** (palette, mobile nav drawer) trap focus and restore it on close.
- [ ] Collapsed icon-only nav items retain accessible names; counts use `tabular-nums`.
