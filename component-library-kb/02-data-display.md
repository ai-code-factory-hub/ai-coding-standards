# 02 · Data Display

Components that present data. Token-driven, a11y-first — see the [shared rules](README.md#shared-rules-apply-to-every-component). Related: [`../design-system-kb/03-components-and-a11y.md`](../design-system-kb/03-components-and-a11y.md) · [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) · [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md).

All numeric columns use `tabular-nums`; numbers/currency/dates are locale-formatted via `Intl.*`.

---

## Table / DataGrid

**Anatomy:** toolbar (optional) → `table` [thead (sortable headers) → tbody (rows, selectable) ] → footer (pagination + row count). Numeric cells right-aligned + `tabular-nums`.

**Variants:** basic · sortable · selectable (checkbox column) · paginated · with sticky header · density (comfortable/compact). **States:** row hover · row selected · sorted column (asc/desc + icon) · loading (skeleton rows) · empty ([EmptyState](03-feedback-and-overlays.md#emptystate)) · error (inline retry).

**Tokens used:** header text `--color-text-muted`, border `--color-border`, row hover `--color-surface`, selected row `--color-primary`/10 tint, `font-mono` for IDs.

**Accessibility:** semantic `<table>` with `<th scope="col">`; sortable header is a `<button>` with `aria-sort="ascending|descending|none"`; selection checkboxes labelled ("Select row N" / "Select all"); caption or `aria-label` names the table; keyboard: Tab to headers/controls, Enter/Space to sort/select.

**Recipe:**
```tsx
type Col<T> = { key: keyof T; header: string; numeric?: boolean; sortable?: boolean; render?: (row: T) => React.ReactNode };

export function DataTable<T extends { id: string }>({ columns, rows, sort, onSort, loading }: {
  columns: Col<T>[]; rows: T[]; sort?: { key: keyof T; dir: "asc" | "desc" }; onSort?: (k: keyof T) => void; loading?: boolean;
}) {
  return (
    <table className="w-full border-collapse text-sm">
      <thead>
        <tr className="border-b border-[--color-border] text-left">
          {columns.map((c) => (
            <th key={String(c.key)} scope="col"
              aria-sort={sort?.key === c.key ? (sort.dir === "asc" ? "ascending" : "descending") : "none"}
              className={cn("px-3 py-2 text-xs font-medium text-[--color-text-muted]", c.numeric && "text-right")}>
              {c.sortable ? (
                <button onClick={() => onSort?.(c.key)} className="inline-flex items-center gap-1 hover:text-[--color-text] focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30 rounded">
                  {c.header}<span aria-hidden>{sort?.key === c.key ? (sort.dir === "asc" ? "▲" : "▼") : "↕"}</span>
                </button>
              ) : c.header}
            </th>
          ))}
        </tr>
      </thead>
      <tbody>
        {loading
          ? Array.from({ length: 5 }).map((_, i) => (
              <tr key={i} className="border-b border-[--color-border]">
                {columns.map((c) => <td key={String(c.key)} className="px-3 py-2"><div className="h-4 w-full animate-pulse rounded bg-[--color-surface] motion-reduce:animate-none" /></td>)}
              </tr>
            ))
          : rows.map((row) => (
              <tr key={row.id} className="border-b border-[--color-border] hover:bg-[--color-surface]">
                {columns.map((c) => (
                  <td key={String(c.key)} className={cn("px-3 py-2 text-[--color-text]", c.numeric && "text-right tabular-nums")}>
                    {c.render ? c.render(row) : String(row[c.key])}
                  </td>
                ))}
              </tr>
            ))}
      </tbody>
    </table>
  );
}
```
> Pair with [Pagination](04-navigation.md#pagination) in the footer and [EmptyState](03-feedback-and-overlays.md#emptystate) when `rows.length === 0`.

---

## List

**Anatomy:** `ul` of rows, each `[leading (avatar/icon)] [primary + secondary text] [trailing (meta/action)]`. **Variants:** simple · interactive (links/buttons) · with dividers · with sections. **States:** default · hover · focus · selected · disabled · loading (skeleton) · empty.

**Tokens used:** divider `--color-border`, hover `--color-surface`, secondary text `--color-text-muted`.

**Accessibility:** `<ul>`/`<li>`; interactive rows are `<a>`/`<button>` (not click handlers on `<li>`); selected uses `aria-current` or `aria-selected`; keyboard focusable.

**Recipe:**
```tsx
export function List({ items }: { items: { id: string; title: string; subtitle?: string; href?: string; meta?: React.ReactNode }[] }) {
  return (
    <ul className="divide-y divide-[--color-border]">
      {items.map((it) => {
        const Row = it.href ? "a" : "div";
        return (
          <li key={it.id}>
            <Row {...(it.href ? { href: it.href } : {})}
              className={cn("flex items-center justify-between gap-3 px-3 py-2.5",
                it.href && "hover:bg-[--color-surface] focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30")}>
              <div className="min-w-0">
                <p className="truncate text-sm text-[--color-text]">{it.title}</p>
                {it.subtitle && <p className="truncate text-xs text-[--color-text-muted]">{it.subtitle}</p>}
              </div>
              {it.meta && <div className="shrink-0 text-sm text-[--color-text-muted] tabular-nums">{it.meta}</div>}
            </Row>
          </li>
        );
      })}
    </ul>
  );
}
```

---

## Card

**Anatomy:** container → optional header (title + action) → body → optional footer. **Variants:** standard · interactive (whole-card link) · AI-insight (only gradient/tinted surface — ✨ badge). **States:** default · hover (optional `sm` lift only if interactive) · focus (if interactive).

**Tokens used:** bg `--color-surface`, border `--color-border`, radius `xl`, title `--color-text`. AI: tint from brand `950/50` + `--color-ai-gradient` accent.

**Accessibility:** if the whole card is a link, wrap a single `<a>` and don't nest interactive children; heading uses a real `<h*>`; AI card labels its nature ("AI insight") in text, not color alone.

**Recipe:**
```tsx
export function Card({ title, action, footer, ai, className, children }: {
  title?: string; action?: React.ReactNode; footer?: React.ReactNode; ai?: boolean; className?: string; children: React.ReactNode;
}) {
  return (
    <section className={cn("rounded-xl border p-4",
      ai ? "border-transparent bg-[--color-surface] ring-1 ring-[--color-primary]/20" : "border-[--color-border] bg-[--color-surface]", className)}>
      {(title || action) && (
        <header className="mb-3 flex items-center justify-between">
          <h3 className="text-sm font-semibold text-[--color-text]">{ai && <span aria-hidden className="mr-1">✨</span>}{title}</h3>
          {action}
        </header>
      )}
      <div className="text-sm text-[--color-text]">{children}</div>
      {footer && <footer className="mt-3 border-t border-[--color-border] pt-3 text-sm text-[--color-text-muted]">{footer}</footer>}
    </section>
  );
}
```

---

## StatCard / KPI

**Anatomy:** `label · big value (tabular-nums) · delta (▲/▼ + %) · optional sparkline/icon`. **Variants:** plain · with trend · with target/bullet · AI-flagged (anomaly). **States:** default · loading (skeleton) · empty ("—") · positive/negative delta (icon + color, never color-alone).

**Tokens used:** value `--color-text`, label `--color-text-muted`, positive `--color-success`, negative `--color-danger`, large number weight `300`.

**Accessibility:** value has an accessible label combining label + value + trend direction in words ("Revenue, ₹1,20,000, up 8%"); delta arrow is `aria-hidden`, direction stated in text.

**Recipe:**
```tsx
export function StatCard({ label, value, deltaPct, locale = "en-IN" }: { label: string; value: number; deltaPct?: number; locale?: string }) {
  const up = (deltaPct ?? 0) >= 0;
  const formatted = new Intl.NumberFormat(locale).format(value);
  return (
    <div className="rounded-xl border border-[--color-border] bg-[--color-surface] p-4">
      <p className="text-xs text-[--color-text-muted]">{label}</p>
      <p className="mt-1 text-3xl font-light tabular-nums text-[--color-text]">{formatted}</p>
      {deltaPct != null && (
        <p className={cn("mt-1 flex items-center gap-1 text-xs tabular-nums", up ? "text-[--color-success]" : "text-[--color-danger]")}>
          <span aria-hidden>{up ? "▲" : "▼"}</span>
          <span>{up ? "up" : "down"} {Math.abs(deltaPct)}%</span>
        </p>
      )}
    </div>
  );
}
```

---

## Badge / Tag

**Anatomy:** pill = `[optional icon] label [optional remove ×]`. **Variants:** neutral · success · warning · danger · info · AI. Removable (tag) variant adds a close button. **States:** default · (removable) hover/focus on ×.

**Tokens used:** semantic color per status (`--color-success` etc.) on a tinted bg; radius `full`. **Status = fixed semantic color + icon + text label — never color-alone.**

**Accessibility:** status badge conveys meaning in **text**, not just color; remove button has `aria-label="Remove {label}"`; if the badge is purely decorative echo of adjacent text, mark `aria-hidden`.

**Recipe:**
```tsx
const tones = {
  neutral: "bg-[--color-surface] text-[--color-text] ring-[--color-border]",
  success: "bg-[--color-success]/10 text-[--color-success] ring-[--color-success]/30",
  warning: "bg-[--color-warning]/10 text-[--color-warning] ring-[--color-warning]/30",
  danger: "bg-[--color-danger]/10 text-[--color-danger] ring-[--color-danger]/30",
  info: "bg-[--color-info]/10 text-[--color-info] ring-[--color-info]/30",
  ai: "bg-[image:--color-ai-gradient] text-white ring-transparent",
} as const;
const icons = { success: "✓", warning: "!", danger: "×", info: "i", ai: "✨", neutral: "" };

export function Badge({ tone = "neutral", children, onRemove }: { tone?: keyof typeof tones; children: React.ReactNode; onRemove?: () => void }) {
  return (
    <span className={cn("inline-flex items-center gap-1 rounded-full px-2 py-0.5 text-xs font-medium ring-1 ring-inset", tones[tone])}>
      {icons[tone] && <span aria-hidden>{icons[tone]}</span>}
      {children}
      {onRemove && <button onClick={onRemove} aria-label="Remove" className="ml-0.5 rounded focus-visible:outline-none focus-visible:ring-[2px] focus-visible:ring-[--color-primary]/30">×</button>}
    </span>
  );
}
```

---

## Avatar

**Anatomy:** circle with image, or initials fallback, or icon; optional presence dot / badge. **Variants:** image · initials · icon · group (stacked). Sizes xs–xl. **States:** default · loading (skeleton) · image-error (falls back to initials).

**Tokens used:** radius `full`; fallback bg may use `--color-accent` (warmth moment, ≤5%) with initials kept ≥4.5:1; ring `--color-bg` for stacked groups.

**Accessibility:** meaningful `alt` (person's name) on images; initials fallback has `role="img" aria-label={name}`; presence dot has a text alternative.

**Recipe:**
```tsx
export function Avatar({ name, src, size = 36 }: { name: string; src?: string; size?: number }) {
  const [err, setErr] = React.useState(false);
  const initials = name.split(" ").map((w) => w[0]).slice(0, 2).join("").toUpperCase();
  return src && !err ? (
    <img src={src} alt={name} width={size} height={size} onError={() => setErr(true)} className="rounded-full object-cover" style={{ width: size, height: size }} />
  ) : (
    <span role="img" aria-label={name} style={{ width: size, height: size }}
      className="inline-grid place-items-center rounded-full bg-[--color-accent] text-xs font-medium text-white">
      {initials}
    </span>
  );
}
```

---

## Tooltip

**Anatomy:** trigger + floating label. Supplementary info only — never the sole source of essential info. **Variants:** top/right/bottom/left. **States:** hidden · visible (hover/focus) · reduced-motion (no fade).

**Tokens used:** bg `--color-text` (inverse) / text `--color-bg`, radius `md`, shadow `md`.

**Accessibility:** `role="tooltip"`; opens on hover **and** keyboard focus; dismissable with Esc; associated to trigger via `aria-describedby`; not used for interactive content.

**Recipe:**
```tsx
import * as Tp from "@radix-ui/react-tooltip";

export function Tooltip({ label, children }: { label: string; children: React.ReactNode }) {
  return (
    <Tp.Provider delayDuration={200}>
      <Tp.Root>
        <Tp.Trigger asChild>{children}</Tp.Trigger>
        <Tp.Portal>
          <Tp.Content sideOffset={6} role="tooltip"
            className="rounded-md bg-[--color-text] px-2 py-1 text-xs text-[--color-bg] shadow-md motion-reduce:animate-none">
            {label}
            <Tp.Arrow className="fill-[--color-text]" />
          </Tp.Content>
        </Tp.Portal>
      </Tp.Root>
    </Tp.Provider>
  );
}
```

---

## DescriptionList

**Anatomy:** `dl` of `dt` (label) / `dd` (value) pairs, key-value layout for detail views. **Variants:** two-column (label/value) · stacked · with dividers. **States:** default · empty value ("—") · loading (skeleton) · copyable value.

**Tokens used:** term `--color-text-muted`, value `--color-text`, divider `--color-border`, `font-mono` for IDs/codes.

**Accessibility:** semantic `<dl>/<dt>/<dd>`; empty values rendered as "—" (not blank); copy buttons labelled.

**Recipe:**
```tsx
export function DescriptionList({ items }: { items: { term: string; value?: React.ReactNode; mono?: boolean }[] }) {
  return (
    <dl className="divide-y divide-[--color-border]">
      {items.map((it) => (
        <div key={it.term} className="grid grid-cols-3 gap-3 py-2">
          <dt className="text-sm text-[--color-text-muted]">{it.term}</dt>
          <dd className={cn("col-span-2 text-sm text-[--color-text]", it.mono && "font-mono")}>{it.value ?? "—"}</dd>
        </div>
      ))}
    </dl>
  );
}
```

---

## Acceptance checklist

- [ ] **Token-driven:** no raw hex; semantic tokens for every color/border/radius.
- [ ] **All states:** hover / focus / selected / loading (skeleton) / empty / error covered where relevant — no blank areas.
- [ ] **Tabular numerals** on every numeric column; numbers/currency/dates formatted via `Intl.*`; numeric columns right-aligned.
- [ ] **Status never color-alone:** badges = semantic color + icon + text label.
- [ ] **A11y:** semantic `<table>/<ul>/<dl>`; sortable headers expose `aria-sort`; images have `alt`/labels; interactive rows are real links/buttons with visible focus rings.
- [ ] Empty and loading states use [EmptyState](03-feedback-and-overlays.md#emptystate) / [Skeleton](03-feedback-and-overlays.md#skeleton).
