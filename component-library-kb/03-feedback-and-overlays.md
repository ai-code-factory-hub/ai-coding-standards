# 03 · Feedback & Overlays

Transient messages, overlays, and status feedback. Token-driven, a11y-first — see the [shared rules](README.md#shared-rules-apply-to-every-component). Related: [`../design-system-kb/03-components-and-a11y.md`](../design-system-kb/03-components-and-a11y.md) · [`../standards-kb/11-frontend-ux.md`](../standards-kb/11-frontend-ux.md) · [`../standards-kb/13-i18n-accessibility.md`](../standards-kb/13-i18n-accessibility.md).

Overlays trap focus, restore it on close, close on Esc, and honor `prefers-reduced-motion`. Status is semantic color + icon + text — never color-alone.

---

## Toast

**Anatomy:** `[icon] title + optional description [optional action] [dismiss ×]`, stacked in a viewport corner, auto-dismiss timer. **Variants:** success · warning · danger · info · AI · with-action (undo). **States:** entering · visible · action-hover/focus · dismissing · paused-on-hover.

**Tokens used:** semantic accent per tone (`--color-success` etc.), surface `--color-surface`, border `--color-border`, shadow `md`, motion `normal 200ms`.

**Accessibility:** viewport is `aria-live="polite"` (danger → `assertive`) and `role="status"`/`"alert"`; auto-dismiss pauses on hover/focus; dismiss button labelled; never rely on color alone (icon + text).

**Recipe:**
```tsx
import * as T from "@radix-ui/react-toast";
const toneIcon = { success: "✓", warning: "!", danger: "×", info: "i", ai: "✨" } as const;

export function Toast({ open, onOpenChange, tone = "info", title, description, action }: {
  open: boolean; onOpenChange: (o: boolean) => void; tone?: keyof typeof toneIcon; title: string; description?: string; action?: React.ReactNode;
}) {
  return (
    <T.Root open={open} onOpenChange={onOpenChange}
      className="flex items-start gap-2 rounded-lg border border-[--color-border] bg-[--color-surface] p-3 shadow-md motion-reduce:animate-none">
      <span aria-hidden className={cn("mt-0.5", `text-[--color-${tone}]`)}>{toneIcon[tone]}</span>
      <div className="min-w-0 flex-1">
        <T.Title className="text-sm font-medium text-[--color-text]">{title}</T.Title>
        {description && <T.Description className="text-xs text-[--color-text-muted]">{description}</T.Description>}
      </div>
      {action && <T.Action asChild altText="Action">{action}</T.Action>}
      <T.Close aria-label="Dismiss" className="text-[--color-text-muted] hover:text-[--color-text]">×</T.Close>
    </T.Root>
  );
}
// Mount once near root: <T.Provider><...toasts...><T.Viewport className="fixed bottom-4 right-4 flex w-80 flex-col gap-2" /></T.Provider>
```

---

## Alert / Banner

**Anatomy:** `[icon] title + body [optional actions] [optional dismiss]`. Inline (contextual) or page-level banner. **Variants:** info · success · warning · danger · AI. Dismissible / persistent. **States:** default · dismissible-hover · dismissed.

**Tokens used:** tinted bg from semantic color `/10`, border/icon semantic color, text `--color-text`.

**Accessibility:** `role="alert"` for urgent/danger (assertive), `role="status"` for informational; icon `aria-hidden`, meaning in text; dismiss labelled; keep readable at 200% zoom.

**Recipe:**
```tsx
const alertTones = {
  info: "bg-[--color-info]/10 border-[--color-info]/30 text-[--color-info]",
  success: "bg-[--color-success]/10 border-[--color-success]/30 text-[--color-success]",
  warning: "bg-[--color-warning]/10 border-[--color-warning]/30 text-[--color-warning]",
  danger: "bg-[--color-danger]/10 border-[--color-danger]/30 text-[--color-danger]",
} as const;

export function Alert({ tone = "info", title, children, urgent }: { tone?: keyof typeof alertTones; title: string; children?: React.ReactNode; urgent?: boolean }) {
  return (
    <div role={urgent ? "alert" : "status"} className={cn("flex gap-2 rounded-lg border p-3", alertTones[tone])}>
      <span aria-hidden>{{ info: "i", success: "✓", warning: "!", danger: "▲" }[tone]}</span>
      <div>
        <p className="text-sm font-medium">{title}</p>
        {children && <div className="mt-0.5 text-sm text-[--color-text]">{children}</div>}
      </div>
    </div>
  );
}
```

---

## Modal / Dialog

**Anatomy:** overlay scrim → panel (`header: title + close · body · footer: actions`). **Variants:** default · scrollable body · form dialog · sizes sm/md/lg. **States:** closed · opening · open (focus trapped) · closing. Reduced-motion: instant.

**Tokens used:** scrim `--color-text`/40 alpha, panel `--color-surface` + `--color-border`, radius `xl`, shadow `lg`, motion `normal 200ms`.

**Accessibility:** `role="dialog" aria-modal="true"` with `aria-labelledby` (title) + `aria-describedby`; focus moves in on open, is **trapped**, returns to the trigger on close; Esc closes; scrim click closes (non-destructive); background `inert`.

**Recipe:**
```tsx
import * as D from "@radix-ui/react-dialog";

export function Modal({ open, onOpenChange, title, description, footer, children }: {
  open: boolean; onOpenChange: (o: boolean) => void; title: string; description?: string; footer?: React.ReactNode; children: React.ReactNode;
}) {
  return (
    <D.Root open={open} onOpenChange={onOpenChange}>
      <D.Portal>
        <D.Overlay className="fixed inset-0 bg-[--color-text]/40 motion-reduce:animate-none" />
        <D.Content aria-describedby={description ? "modal-desc" : undefined}
          className="fixed left-1/2 top-1/2 w-[92vw] max-w-lg -translate-x-1/2 -translate-y-1/2 rounded-xl border border-[--color-border] bg-[--color-surface] p-5 shadow-lg motion-reduce:animate-none">
          <div className="mb-3 flex items-start justify-between">
            <D.Title className="text-base font-semibold text-[--color-text]">{title}</D.Title>
            <D.Close aria-label="Close" className="text-[--color-text-muted] hover:text-[--color-text] focus-visible:outline-none focus-visible:ring-[3px] focus-visible:ring-[--color-primary]/30 rounded">×</D.Close>
          </div>
          {description && <D.Description id="modal-desc" className="mb-3 text-sm text-[--color-text-muted]">{description}</D.Description>}
          <div className="text-sm text-[--color-text]">{children}</div>
          {footer && <div className="mt-5 flex justify-end gap-2">{footer}</div>}
        </D.Content>
      </D.Portal>
    </D.Root>
  );
}
```

---

## Drawer / Sheet

**Anatomy:** edge-anchored panel (right/left/bottom) + scrim; header/body/footer like Modal. **Variants:** side (right/left) · bottom sheet (mobile) · sizes. **States:** closed/opening/open(trapped)/closing.

**Tokens used:** same as Modal; slides from edge; motion `normal 200ms`, reduced-motion instant.

**Accessibility:** identical semantics to Modal (`role="dialog" aria-modal`, focus trap + restore, Esc close); swipe-to-dismiss on touch has a keyboard/button equivalent.

**Recipe:**
```tsx
import * as D from "@radix-ui/react-dialog";
// Reuse Dialog primitive; override Content position/animation for the edge.
export function Drawer({ open, onOpenChange, title, side = "right", children }: {
  open: boolean; onOpenChange: (o: boolean) => void; title: string; side?: "right" | "left"; children: React.ReactNode;
}) {
  return (
    <D.Root open={open} onOpenChange={onOpenChange}>
      <D.Portal>
        <D.Overlay className="fixed inset-0 bg-[--color-text]/40" />
        <D.Content className={cn("fixed inset-y-0 w-[90vw] max-w-md border-[--color-border] bg-[--color-surface] p-5 shadow-lg motion-reduce:animate-none",
          side === "right" ? "right-0 border-l" : "left-0 border-r")}>
          <div className="mb-3 flex items-center justify-between">
            <D.Title className="text-base font-semibold text-[--color-text]">{title}</D.Title>
            <D.Close aria-label="Close" className="text-[--color-text-muted] hover:text-[--color-text]">×</D.Close>
          </div>
          <div className="text-sm text-[--color-text]">{children}</div>
        </D.Content>
      </D.Portal>
    </D.Root>
  );
}
```

---

## Skeleton

**Anatomy:** shimmer placeholder blocks mirroring the shape of the incoming content. **Variants:** text line · circle (avatar) · rect (card/media) · table row. **States:** shimmering (motion) · static (reduced-motion).

**Tokens used:** base `--color-surface`, shimmer highlight `--color-border`, motion `skeleton 1500ms`.

**Accessibility:** container marked `aria-hidden` (decorative) with an adjacent `aria-live` "Loading…" for SR users; shimmer disabled under `prefers-reduced-motion`.

**Recipe:**
```tsx
export function Skeleton({ className }: { className?: string }) {
  return <div aria-hidden className={cn("animate-pulse rounded bg-[--color-surface] motion-reduce:animate-none", className)} />;
}
// Usage: <div role="status" aria-live="polite"><span className="sr-only">Loading</span><Skeleton className="h-4 w-40" /></div>
```

---

## EmptyState

**Anatomy:** `illustration/icon · title · description · primary action(s)`. Shown for no-data, no-results (filtered), and first-run. **Variants:** no-data · no-search-results (offer clear-filters) · error (offer retry) · restricted (no permission). **States:** default · with-action.

**Tokens used:** icon/text-muted `--color-text-muted`, title `--color-text`, uses [Button](01-forms.md#button) for the action.

**Accessibility:** real heading; action is a focusable button; not conveyed by illustration alone — the message is in text.

**Recipe:**
```tsx
export function EmptyState({ icon, title, description, action }: { icon?: React.ReactNode; title: string; description?: string; action?: React.ReactNode }) {
  return (
    <div className="flex flex-col items-center gap-2 py-12 text-center">
      {icon && <div className="text-[--color-text-muted]" aria-hidden>{icon}</div>}
      <h3 className="text-sm font-semibold text-[--color-text]">{title}</h3>
      {description && <p className="max-w-sm text-sm text-[--color-text-muted]">{description}</p>}
      {action && <div className="mt-2">{action}</div>}
    </div>
  );
}
```

---

## ConfirmDialog

**Anatomy:** Modal specialization → title (question) · body (consequence) · footer (cancel + confirm). Destructive confirm uses the `danger` Button. **Variants:** default · destructive · with typed-confirmation (type resource name). **States:** idle · confirming (loading on confirm button) · error.

**Tokens used:** Modal tokens; confirm button `--color-danger` when destructive.

**Accessibility:** `role="alertdialog"` with `aria-labelledby`/`aria-describedby`; initial focus on the **safe** action (Cancel) for destructive ops; Esc = cancel; consequence stated in text.

**Recipe:**
```tsx
export function ConfirmDialog({ open, onOpenChange, title, body, confirmLabel = "Confirm", destructive, loading, onConfirm }: {
  open: boolean; onOpenChange: (o: boolean) => void; title: string; body: string; confirmLabel?: string; destructive?: boolean; loading?: boolean; onConfirm: () => void;
}) {
  return (
    <Modal open={open} onOpenChange={onOpenChange} title={title} description={body}
      footer={<>
        <Button variant="secondary" onClick={() => onOpenChange(false)}>Cancel</Button>
        <Button variant={destructive ? "danger" : "primary"} loading={loading} onClick={onConfirm}>{confirmLabel}</Button>
      </>}>
      {null}
    </Modal>
  );
}
// Note: set the underlying Dialog Content role="alertdialog" and focus Cancel first for destructive actions.
```

---

## ProgressBar / Spinner

**Anatomy:** ProgressBar = track + fill (+ optional %). Spinner = rotating indicator with a label. **Variants:** ProgressBar determinate / indeterminate; Spinner sizes sm/md/lg; inline vs. block. **States:** in-progress · complete (100%) · error (danger fill).

**Tokens used:** track `--color-surface`, fill `--color-primary` (error → `--color-danger`), motion `skeleton 1500ms` for indeterminate.

**Accessibility:** ProgressBar `role="progressbar"` with `aria-valuenow/min/max` (omit `valuenow` for indeterminate) + accessible name; Spinner has `role="status"` + visually-hidden label; both respect reduced-motion.

**Recipe:**
```tsx
export function ProgressBar({ value, label }: { value?: number; label: string }) {
  const indeterminate = value == null;
  return (
    <div role="progressbar" aria-label={label} aria-valuenow={value} aria-valuemin={0} aria-valuemax={100}
      className="h-2 w-full overflow-hidden rounded-full bg-[--color-surface]">
      <div className={cn("h-full rounded-full bg-[--color-primary]", indeterminate && "w-1/3 animate-pulse motion-reduce:animate-none")}
        style={indeterminate ? undefined : { width: `${value}%` }} />
    </div>
  );
}

export function Spinner({ label = "Loading" }: { label?: string }) {
  return (
    <span role="status" className="inline-flex items-center gap-2 text-[--color-text-muted]">
      <span className="size-4 animate-spin rounded-full border-2 border-[--color-border] border-t-[--color-primary] motion-reduce:animate-none" aria-hidden />
      <span className="sr-only">{label}</span>
    </span>
  );
}
```

---

## Acceptance checklist

- [ ] **Token-driven:** no raw hex; semantic tokens for scrims, fills, tints, borders.
- [ ] **All states:** entering/visible/dismissing for toasts; open/closing + focus-trap for overlays; determinate/indeterminate/error for progress.
- [ ] **Overlays:** `role="dialog"`/`"alertdialog"` + `aria-modal`, `aria-labelledby`/`describedby`, focus trapped on open and **restored to trigger** on close, Esc closes, background inert.
- [ ] **Live regions:** toasts/alerts use `aria-live` (polite, or assertive for danger); loading has an SR-announced status.
- [ ] **Status never color-alone:** icon + text on every tone.
- [ ] **Motion:** all animation gated behind `motion-safe:` / disabled under `prefers-reduced-motion`.
- [ ] Every async surface has a designed loading ([Skeleton](#skeleton)/[Spinner](#progressbar--spinner)), empty ([EmptyState](#emptystate)), and error state.
