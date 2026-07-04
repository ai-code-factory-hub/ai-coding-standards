# 04 · Frontend Performance

**Purpose:** hit the Core Web Vitals and bundle budgets on real, mid-range devices —
bundle splitting, list virtualization, cheap renders, image/asset discipline, and CDN caching.
Implements the frontend rules of [../standards-kb/03-performance-scalability.md](../standards-kb/03-performance-scalability.md)
and complements [../standards-kb/11-frontend-ux.md](../standards-kb/11-frontend-ux.md).

## Budgets (recap)

- **[MUST]** Initial JS bundle **< 250 KB gzipped**; **LCP < 2.5 s**, **FCP < 1.8 s**,
  **INP < 200 ms**, **CLS < 0.1** on a mid-range device / throttled 4G. Enforced in CI
  (see [05 · Profiling & Benchmarks](05-profiling-and-benchmarks.md)).

## Bundle splitting & lazy routes

- **[MUST]** **Route-level code splitting** — each route loads its own chunk; the initial bundle
  contains only the shell + first view. Heavy, rarely-used components (rich editors, charts,
  PDF/export viewers) are **lazy-loaded on demand**.

  ```
  # Eager: everything in the initial bundle -> blows the 250 KB budget
  import ReportBuilder from './ReportBuilder'      # BAD

  # Lazy: separate chunk, fetched only when the route/interaction needs it
  const ReportBuilder = lazy(() => import('./ReportBuilder'))
  <Route path="/reports/build" element={<Suspense fallback={<Skel/>}><ReportBuilder/></Suspense>}/>
  ```

- **[MUST]** **Tree-shake** and import narrowly (`import { fn } from 'lib/fn'`, not the whole lib).
  Audit the bundle; drop or replace heavyweight dependencies (date/icon/utility mega-libs).
- **[SHOULD]** **Prefetch** the next likely chunk on intent (link hover / viewport) so navigation
  feels instant without inflating the initial load.
- **[SHOULD]** Split **vendor vs app** chunks for long-term caching; ship modern ES to modern
  browsers (differential/`module` builds) to cut transpile bloat.

## Virtualization & pagination

- **[MUST]** **Never render an unbounded list.** Server-paginate (keyset — see
  [02 · Query & Statement](02-query-and-statement.md)) and **windowed/virtualized** rendering so
  only visible rows mount. A 10,000-row table renders ~30 DOM nodes, not 10,000.
- **[SHOULD]** Use infinite scroll or explicit paging with a stable cursor; keep a fixed row height
  (or measured) so scroll math and CLS stay stable.

## Render performance

- **[MUST]** Prevent needless re-renders: stable references, memoized components/derived values,
  and **not** creating new object/array/function props inline on hot paths. This is the main INP
  lever — a heavy re-render on each keystroke/click blows the 200 ms interaction budget.
- **[MUST]** Keep the **main thread free during interactions**: break long tasks (> 50 ms), defer
  non-urgent work (idle callback / transition), and debounce/throttle high-frequency handlers
  (input, scroll, resize).
- **[SHOULD]** Offload heavy compute (parsing, CSV, crypto, large diffing) to a **web worker** so
  the UI thread stays responsive.
- **[SHOULD]** Show skeletons/optimistic UI immediately; render above-the-fold first, hydrate/defer
  the rest.

## Core Web Vitals — what moves each

- **FCP / LCP** — shrink the critical path: small initial JS, server-render or stream the shell,
  preload the LCP image/font, inline critical CSS, avoid render-blocking resources.
- **INP** — cheap event handlers, no long tasks, yield to the main thread; the biggest wins are
  render discipline + worker offload above.
- **CLS** — **reserve space**: width/height (or aspect-ratio) on every image/embed, reserved slots
  for async content and ads, no layout-shifting late-loaded fonts (`font-display: optional/swap` +
  metric-matched fallback).
- **[MUST]** Measure vitals from the **field** (RUM), not only lab — segment by device class and
  tenant.

## Image & asset optimization

- **[MUST]** Serve **modern formats** (AVIF/WebP) with responsive `srcset`/`sizes`; never ship a
  2000 px image into a 320 px slot. **Lazy-load** below-the-fold images; **eager + high priority**
  for the LCP image only.
- **[MUST]** Subset and self-host/preload fonts; avoid loading multiple weights you don't use.
- **[SHOULD]** Inline tiny SVG icons; sprite the rest. Compress everything at build time.

## Caching & CDN

- **[MUST]** Serve static assets from a **CDN** with **content-hashed filenames** and
  `Cache-Control: immutable, max-age=1y`; the HTML entry is short-cached/revalidated so new hashes
  roll out cleanly.
- **[SHOULD]** Cache API GETs with `ETag`/`Cache-Control` where staleness is tolerable; use
  stale-while-revalidate at the edge for hot, low-churn responses. **Tenant-scope** any shared
  cache keys/edge rules.
- **[SHOULD]** Enable HTTP/2+ and compression (Brotli) at the edge; preconnect to critical origins.

## Minimal fields per screen

- **[MUST]** **Progressive disclosure** — request and render only the fields the screen needs
  (config-driven per tenant/role, per Standard 03). Fewer fields → smaller payloads, cheaper
  renders, faster LCP. Pair with narrow-column API projections from
  [02 · Query & Statement](02-query-and-statement.md).

> **Example (multi-tenant LIS worklist):** the technician worklist showed 40 columns and rendered
> every one of 8,000 samples. Cutting to the 6 role-relevant fields (config-driven), keyset paging,
> and row virtualization dropped the initial payload ~85% and brought INP from ~450 ms to < 120 ms.

> **Example (e-commerce catalog):** the LCP hero shipped a 1.8 MB PNG with no dimensions —
> LCP 4.1 s, visible CLS. AVIF + responsive `srcset` + reserved aspect-ratio + preload took LCP
> under 2.5 s and CLS to ~0.

## Acceptance checklist

- [ ] Initial JS < 250 KB gzipped; routes code-split; heavy components lazy-loaded and tree-shaken.
- [ ] All long lists server-paginated (keyset) and virtualized — no unbounded renders.
- [ ] Re-renders minimized; long tasks broken up; heavy compute offloaded to workers; INP < 200 ms.
- [ ] LCP < 2.5 s / FCP < 1.8 s / CLS < 0.1; space reserved for images/embeds/async content.
- [ ] Images in modern formats with responsive srcset + lazy-load; fonts subset/preloaded.
- [ ] Static assets content-hashed + immutable on a CDN; HTML revalidated; compression on.
- [ ] Screens use progressive disclosure — only role/tenant-relevant fields fetched and rendered.
- [ ] Web Vitals measured in the field (RUM), segmented by device and tenant.
