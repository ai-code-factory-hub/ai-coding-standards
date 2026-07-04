# 01 · Forms

Form primitives. Every one is token-driven and a11y-first — see the [shared rules](README.md#shared-rules-apply-to-every-component). Related: [`../design-system-kb/03-components-and-a11y.md`](../design-system-kb/03-components-and-a11y.md) · [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) · [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md).

Conventions: `cn()` = class-merge helper; focus ring = `focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30`; buttons `rounded-lg`, inputs `rounded-md`.

---

## Button

**Anatomy:** `[optional leading icon] label [optional trailing icon / spinner]`. Icon-only variant carries an `aria-label`.

**Variants:** `primary` (one per screen) · `secondary` (bordered) · `ghost` (text-only) · `danger` (destructive confirm) · `ai` (indigo→purple gradient + ✨ — the only gradient). Sizes: `sm` (h-8) · `md` (h-9, default) · `lg` (h-11, ≥44px touch).

**States:** default · hover (surface tint / darker step) · focus (mandatory ring) · active (pressed step) · disabled (`~300` tint, `cursor-not-allowed`, `aria-disabled`) · loading (spinner replaces leading icon, `aria-busy`, disabled interaction, label stays).

**Tokens used:** `--color-primary`, `--color-danger`, `--color-border`, `--color-text`, `--color-surface`, `--color-ai-gradient`, radius `lg`, focus ring token, motion `fast 150ms`.

**Accessibility:** native `<button type=…>`; icon-only requires `aria-label`; loading sets `aria-busy="true"` and keeps the button focusable-but-inert; visible focus ring never removed; hit area ≥ 44×44 for `lg`/mobile.

**Recipe:**
```tsx
import { Loader2 } from "lucide-react";
import { cn } from "@/lib/cn";

type Variant = "primary" | "secondary" | "ghost" | "danger" | "ai";

const variants: Record<Variant, string> = {
  primary: "bg-[--color-primary] text-white hover:brightness-95 active:brightness-90",
  secondary: "border border-[--color-border] bg-transparent text-[--color-text] hover:bg-[--color-surface]",
  ghost: "text-[--color-primary] hover:bg-[--color-surface]",
  danger: "bg-[--color-danger] text-white hover:brightness-95",
  ai: "bg-[image:--color-ai-gradient] text-white hover:brightness-105",
};

export function Button({
  variant = "primary", loading, disabled, className, children, ...props
}: React.ButtonHTMLAttributes<HTMLButtonElement> & { variant?: Variant; loading?: boolean }) {
  return (
    <button
      disabled={disabled || loading}
      aria-busy={loading || undefined}
      className={cn(
        "inline-flex h-9 items-center justify-center gap-2 rounded-lg px-4 text-sm font-medium",
        "motion-safe:transition-[filter,background-color] motion-safe:duration-150",
        "focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30",
        "disabled:cursor-not-allowed disabled:opacity-60",
        variants[variant], className,
      )}
      {...props}
    >
      {loading && <Loader2 className="size-4 animate-spin motion-reduce:animate-none" aria-hidden />}
      {variant === "ai" && !loading && <span aria-hidden>✨</span>}
      {children}
    </button>
  );
}
```

---

## Input

**Anatomy:** `[optional leading icon] control [optional trailing icon / clear / unit]`. Always paired with a label via [FormField](#formfield-label--error--hint).

**Variants:** text / email / number / password / search. Optional leading icon, trailing addon (unit, clear).

**States:** default · hover (border darken) · focus (border→primary + ring) · disabled (`--color-surface`, muted text) · error (`--color-danger` border + `aria-invalid`) · read-only (`readOnly`, no focus affordance, muted).

**Tokens used:** `--color-bg`, `--color-border`, `--color-text`, `--color-text-muted` (placeholder), `--color-primary` (focus), `--color-danger` (error), radius `md`.

**Accessibility:** programmatic label (`htmlFor`/`id`); `aria-invalid` + `aria-describedby` pointing at the error/hint; `inputMode`/`type` set for correct mobile keyboards; numeric inputs use `tabular-nums`.

**Recipe:**
```tsx
export const Input = React.forwardRef<HTMLInputElement, React.InputHTMLAttributes<HTMLInputElement> & { invalid?: boolean }>(
  ({ invalid, className, ...props }, ref) => (
    <input
      ref={ref}
      aria-invalid={invalid || undefined}
      className={cn(
        "h-9 w-full rounded-md border bg-[--color-bg] px-3 text-sm text-[--color-text]",
        "placeholder:text-[--color-text-muted]",
        "motion-safe:transition-colors focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30",
        invalid ? "border-[--color-danger] focus-visible:border-[--color-danger]" : "border-[--color-border] focus-visible:border-[--color-primary]",
        "read-only:bg-[--color-surface] read-only:text-[--color-text-muted]",
        "disabled:cursor-not-allowed disabled:bg-[--color-surface] disabled:text-[--color-text-muted]",
        className,
      )}
      {...props}
    />
  ),
);
Input.displayName = "Input";
```

---

## Textarea

**Anatomy:** multiline control + optional character counter (announced with `aria-live="polite"`).

**Variants:** fixed rows · auto-grow. **States:** same as Input (default/hover/focus/disabled/error/read-only).

**Tokens used:** identical to Input.

**Accessibility:** labelled; `aria-describedby` for counter/hint; counter is polite live region; respects resize (`resize-y`).

**Recipe:**
```tsx
export function Textarea({ invalid, className, ...props }: React.TextareaHTMLAttributes<HTMLTextAreaElement> & { invalid?: boolean }) {
  return (
    <textarea
      aria-invalid={invalid || undefined}
      className={cn(
        "min-h-20 w-full resize-y rounded-md border bg-[--color-bg] p-3 text-sm text-[--color-text]",
        "placeholder:text-[--color-text-muted] motion-safe:transition-colors",
        "focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30",
        invalid ? "border-[--color-danger]" : "border-[--color-border] focus-visible:border-[--color-primary]",
        className,
      )}
      {...props}
    />
  );
}
```

---

## Select

**Anatomy:** trigger (value + chevron) → listbox popover → options (with check on selected). Prefer a Radix/Headless Select for full keyboard + ARIA.

**Variants:** single · with groups · with descriptions. **States:** default/hover/focus/open/disabled/error/read-only; option: hover/active(highlighted)/selected/disabled.

**Tokens used:** trigger like Input; popover `--color-surface` + `--color-border` + shadow `md`; option hover `--color-surface`, selected `--color-primary` accent.

**Accessibility:** trigger `role="combobox" aria-expanded aria-controls`; listbox `role="listbox"`, options `role="option" aria-selected`; full keyboard: ↑/↓ move, Enter/Space select, Esc close, type-ahead; focus returns to trigger on close.

**Recipe:**
```tsx
import * as S from "@radix-ui/react-select";
import { Check, ChevronDown } from "lucide-react";

export function Select({ value, onValueChange, placeholder, children }: {
  value?: string; onValueChange?: (v: string) => void; placeholder?: string; children: React.ReactNode;
}) {
  return (
    <S.Root value={value} onValueChange={onValueChange}>
      <S.Trigger className={cn(
        "inline-flex h-9 w-full items-center justify-between rounded-md border border-[--color-border] bg-[--color-bg] px-3 text-sm text-[--color-text]",
        "focus:outline-none focus:ring-[3px] focus:ring-[--color-primary]/30 focus:border-[--color-primary] data-[disabled]:opacity-60",
      )}>
        <S.Value placeholder={placeholder} />
        <ChevronDown className="size-4 text-[--color-text-muted]" aria-hidden />
      </S.Trigger>
      <S.Portal>
        <S.Content className="rounded-md border border-[--color-border] bg-[--color-surface] p-1 shadow-md">
          <S.Viewport>{children}</S.Viewport>
        </S.Content>
      </S.Portal>
    </S.Root>
  );
}

export function Option({ value, children }: { value: string; children: React.ReactNode }) {
  return (
    <S.Item value={value} className={cn(
      "flex cursor-pointer items-center gap-2 rounded px-2 py-1.5 text-sm text-[--color-text] outline-none",
      "data-[highlighted]:bg-[--color-surface] data-[state=checked]:font-medium data-[disabled]:opacity-50",
    )}>
      <S.ItemIndicator><Check className="size-4 text-[--color-primary]" aria-hidden /></S.ItemIndicator>
      <S.ItemText>{children}</S.ItemText>
    </S.Item>
  );
}
```

---

## Combobox

**Anatomy:** text input (filters) + listbox of matches + optional async loading/empty rows + optional multi-select chips.

**Variants:** single · multi (chips) · async (debounced fetch). **States:** default/focus/open/loading/empty/error/disabled; option hover/active/selected.

**Tokens used:** input tokens + popover tokens; chips use Badge tokens; loading uses Spinner.

**Accessibility:** `role="combobox" aria-autocomplete="list" aria-expanded aria-activedescendant`; listbox/options as in Select; announce result count via `aria-live`; typing filters, ↑/↓ navigate, Enter select, Esc close/clear.

**Recipe:**
```tsx
export function Combobox({ options, value, onChange, loading, query, onQuery }: {
  options: { id: string; label: string }[]; value?: string; onChange: (id: string) => void;
  loading?: boolean; query: string; onQuery: (q: string) => void;
}) {
  const [open, setOpen] = React.useState(false);
  const [active, setActive] = React.useState(0);
  return (
    <div className="relative">
      <Input role="combobox" aria-expanded={open} aria-autocomplete="list" aria-controls="cb-list"
        value={query} onChange={(e) => { onQuery(e.target.value); setOpen(true); }}
        onKeyDown={(e) => {
          if (e.key === "ArrowDown") setActive((a) => Math.min(a + 1, options.length - 1));
          if (e.key === "ArrowUp") setActive((a) => Math.max(a - 1, 0));
          if (e.key === "Enter" && options[active]) { onChange(options[active].id); setOpen(false); }
          if (e.key === "Escape") setOpen(false);
        }} />
      {open && (
        <ul id="cb-list" role="listbox" className="absolute z-10 mt-1 w-full rounded-md border border-[--color-border] bg-[--color-surface] p-1 shadow-md">
          {loading && <li className="px-2 py-1.5 text-sm text-[--color-text-muted]" aria-live="polite">Loading…</li>}
          {!loading && options.length === 0 && <li className="px-2 py-1.5 text-sm text-[--color-text-muted]">No results</li>}
          {options.map((o, i) => (
            <li key={o.id} role="option" aria-selected={o.id === value}
              className={cn("cursor-pointer rounded px-2 py-1.5 text-sm text-[--color-text]", i === active && "bg-[--color-surface]")}
              onMouseEnter={() => setActive(i)} onClick={() => { onChange(o.id); setOpen(false); }}>
              {o.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

---

## Checkbox

**Anatomy:** box (check / indeterminate glyph) + label. **Variants:** single · indeterminate (parent of a group). **States:** unchecked/checked/indeterminate × default/hover/focus/disabled/error.

**Tokens used:** border `--color-border`, checked bg `--color-primary`, focus ring token, error `--color-danger`.

**Accessibility:** native `<input type="checkbox">` or Radix; indeterminate via `aria-checked="mixed"`; label clickable; Space toggles; ≥44px hit area on touch.

**Recipe:**
```tsx
import * as Cb from "@radix-ui/react-checkbox";
import { Check, Minus } from "lucide-react";

export function Checkbox({ id, label, indeterminate, ...props }: Cb.CheckboxProps & { id: string; label: string; indeterminate?: boolean }) {
  return (
    <label htmlFor={id} className="inline-flex cursor-pointer items-center gap-2 py-1 text-sm text-[--color-text]">
      <Cb.Root id={id} checked={indeterminate ? "indeterminate" : props.checked} {...props}
        className={cn("grid size-5 place-items-center rounded border border-[--color-border] bg-[--color-bg]",
          "data-[state=checked]:border-[--color-primary] data-[state=checked]:bg-[--color-primary]",
          "focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30 disabled:opacity-60")}>
        <Cb.Indicator className="text-white">{indeterminate ? <Minus className="size-3.5" /> : <Check className="size-3.5" />}</Cb.Indicator>
      </Cb.Root>
      {label}
    </label>
  );
}
```

---

## Radio / RadioGroup

**Anatomy:** group of mutually-exclusive dots + labels, one selected. **Variants:** vertical · horizontal · card-style. **States:** unselected/selected × default/hover/focus/disabled/error.

**Tokens used:** border `--color-border`, selected `--color-primary`, focus ring, error `--color-danger`.

**Accessibility:** `role="radiogroup"` with an accessible name; roving tabindex; ↑/↓/←/→ move + select; only the checked radio is tabbable.

**Recipe:**
```tsx
import * as R from "@radix-ui/react-radio-group";

export function RadioGroup({ label, value, onValueChange, options }: {
  label: string; value?: string; onValueChange: (v: string) => void; options: { value: string; label: string }[];
}) {
  return (
    <R.Root aria-label={label} value={value} onValueChange={onValueChange} className="flex flex-col gap-2">
      {options.map((o) => (
        <label key={o.value} className="inline-flex cursor-pointer items-center gap-2 text-sm text-[--color-text]">
          <R.Item value={o.value}
            className={cn("grid size-5 place-items-center rounded-full border border-[--color-border] bg-[--color-bg]",
              "data-[state=checked]:border-[--color-primary] focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30 disabled:opacity-60")}>
            <R.Indicator className="size-2.5 rounded-full bg-[--color-primary]" />
          </label>
        </label>
      ))}
    </R.Root>
  );
}
```

---

## Switch

**Anatomy:** track + thumb + optional label. Use for immediate on/off settings (not form submit toggles — use Checkbox there).

**Variants:** with/without label. **States:** off/on × default/hover/focus/disabled/loading (optimistic).

**Tokens used:** off track `--color-border`, on track `--color-primary`, thumb `--color-bg`, focus ring.

**Accessibility:** `role="switch" aria-checked`; Space/Enter toggles; label associated; never color-alone — position of thumb conveys state.

**Recipe:**
```tsx
import * as Sw from "@radix-ui/react-switch";

export function Switch({ id, label, ...props }: Sw.SwitchProps & { id: string; label?: string }) {
  return (
    <div className="inline-flex items-center gap-2">
      <Sw.Root id={id} {...props}
        className={cn("relative h-6 w-11 rounded-full bg-[--color-border] motion-safe:transition-colors",
          "data-[state=checked]:bg-[--color-primary] focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30 disabled:opacity-60")}>
        <Sw.Thumb className="block size-5 translate-x-0.5 rounded-full bg-white motion-safe:transition-transform data-[state=checked]:translate-x-[22px]" />
      </Sw.Root>
      {label && <label htmlFor={id} className="text-sm text-[--color-text]">{label}</label>}
    </div>
  );
}
```

---

## DatePicker

**Anatomy:** Input (localized display) + trigger → calendar popover (month grid, prev/next, today). **Variants:** single date · range · date-time. **States:** default/focus/open/disabled/error/read-only; day cell: default/hover/today/selected/in-range/outside-month/disabled.

**Tokens used:** input tokens; calendar `--color-surface`/`--color-border`/shadow `md`; selected `--color-primary`; today ring `--color-primary`.

**Accessibility:** `Intl.DateTimeFormat` for display + parsing to tenant/user locale & timezone (store UTC — see [`13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md)); calendar is a `grid` with `aria-label`; arrow keys move by day, PageUp/Down by month, Enter selects, Esc closes; selected day `aria-selected`; announce focused date.

**Recipe:**
```tsx
// Wrap react-day-picker (or Radix) — tokens applied via classNames; formatting via Intl.
import { DayPicker } from "react-day-picker";

export function DatePicker({ value, onChange, locale = "en-IN", timeZone }: {
  value?: Date; onChange: (d?: Date) => void; locale?: string; timeZone?: string;
}) {
  const label = value ? new Intl.DateTimeFormat(locale, { dateStyle: "medium", timeZone }).format(value) : "Select date";
  const [open, setOpen] = React.useState(false);
  return (
    <div className="relative">
      <button onClick={() => setOpen((o) => !o)} aria-haspopup="dialog" aria-expanded={open}
        className="h-9 w-full rounded-md border border-[--color-border] bg-[--color-bg] px-3 text-left text-sm text-[--color-text] focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30">
        {label}
      </button>
      {open && (
        <div role="dialog" aria-label="Choose date" className="absolute z-10 mt-1 rounded-md border border-[--color-border] bg-[--color-surface] p-2 shadow-md">
          <DayPicker mode="single" selected={value} onSelect={(d) => { onChange(d); setOpen(false); }}
            classNames={{ selected: "bg-[--color-primary] text-white rounded", today: "ring-1 ring-[--color-primary] rounded" }} />
        </div>
      )}
    </div>
  );
}
```

---

## FormField (label + error + hint)

**Anatomy:** `label [required *] · control (children) · hint (muted) · error (danger + icon)`. The wiring layer that makes every control accessible.

**Variants:** required · optional · with hint · with error · read-only. **States:** default · error (shows message, sets `aria-invalid` on control via `aria-describedby`) · disabled/read-only (dims label).

**Tokens used:** label `--color-text`, hint `--color-text-muted`, error `--color-danger`, required mark `--color-danger`.

**Accessibility:** generates a stable `id`; associates `label htmlFor`; passes `aria-describedby` (hint + error ids) and `aria-invalid` down to the control; error uses **icon + text** (never color-alone) and is `role="alert"` when it appears.

**Recipe:**
```tsx
import { AlertCircle } from "lucide-react";

export function FormField({ label, hint, error, required, children }: {
  label: string; hint?: string; error?: string; required?: boolean;
  children: (a: { id: string; invalid: boolean; describedBy?: string }) => React.ReactNode;
}) {
  const id = React.useId();
  const hintId = hint ? `${id}-hint` : undefined;
  const errId = error ? `${id}-err` : undefined;
  const describedBy = [hintId, errId].filter(Boolean).join(" ") || undefined;
  return (
    <div className="flex flex-col gap-1.5">
      <label htmlFor={id} className="text-sm font-medium text-[--color-text]">
        {label}{required && <span className="ml-0.5 text-[--color-danger]" aria-hidden>*</span>}
      </label>
      {children({ id, invalid: !!error, describedBy })}
      {hint && !error && <p id={hintId} className="text-xs text-[--color-text-muted]">{hint}</p>}
      {error && (
        <p id={errId} role="alert" className="flex items-center gap-1 text-xs text-[--color-danger]">
          <AlertCircle className="size-3.5" aria-hidden /> {error}
        </p>
      )}
    </div>
  );
}

// Usage:
// <FormField label="Email" required error={errors.email} hint="Work email">
//   {({ id, invalid, describedBy }) => <Input id={id} type="email" invalid={invalid} aria-describedby={describedBy} />}
// </FormField>
```

---

## Acceptance checklist

- [ ] **Token-driven:** no raw hex anywhere; every color/radius/spacing maps to a semantic token from [`../design-system-kb/01-design-tokens.md`](../design-system-kb/01-design-tokens.md).
- [ ] **All states** implemented per component (default / hover / focus / active / disabled / loading / error / read-only as relevant).
- [ ] **A11y:** every control has a programmatic label; `aria-invalid` + `aria-describedby` wire errors/hints; visible focus ring on every interactive element; full keyboard operation (Space/Enter/arrows/Esc/type-ahead where relevant); errors are icon + text (never color-alone); touch targets ≥ 44×44px.
- [ ] **i18n:** all strings externalized; dates/numbers via `Intl.*`; UTC stored, locale/timezone displayed.
- [ ] **Motion:** transitions gated behind `motion-safe:` / honor `prefers-reduced-motion`.
- [ ] One primary Button per screen; only the `ai` variant uses a gradient.
