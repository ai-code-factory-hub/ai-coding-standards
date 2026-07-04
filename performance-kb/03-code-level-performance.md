# 03 · Code-Level Performance

**Purpose:** the application-tier patterns that keep CPU, memory, and I/O off the critical path —
hot-path discipline, allocation/GC pressure, streaming, caching tiers, batch I/O, and async
offload. Implements the "stream don't buffer / bound caches / async for slow work" rules of
[../standards-kb/03-performance-scalability.md](../standards-kb/03-performance-scalability.md).

## Optimize the hot path first (measure, don't guess)

- **[MUST]** Optimize only what a profiler proves is hot. Rank by **total time = frequency ×
  per-call cost**; a function called a million times matters more than a slow one called twice.
  See [05 · Profiling & Benchmarks](05-profiling-and-benchmarks.md).
- **[SHOULD]** On the hot path: hoist invariant work out of loops, precompute lookups, replace
  linear scans with hash/index lookups (`O(n)` → `O(1)`), and avoid repeated work across a request
  (compute once, pass down).
- **[MUST NOT]** Micro-optimize cold paths at the cost of readability — reserve intensity for the
  proven hot 5%.

## Allocation & GC pressure

- **[SHOULD]** Reduce allocations on hot paths: reuse buffers/objects (pooling), preallocate slices
  /maps to known size, avoid boxing and needless intermediate collections in tight loops.
- **[SHOULD]** Avoid **allocation storms** from string building and repeated concatenation — use a
  builder/writer and write once.
- **[SHOULD]** In GC runtimes, high allocation rate → frequent GC → tail-latency jitter. Watch GC
  pause p99 and allocation rate as first-class metrics.

  ```
  # Hot loop: allocates a new list + N temp strings every call  -> GC churn
  def format_rows(rows): return [ (r.id + "-" + r.name) for r in rows ]   # BAD on hot path

  # Reuse a buffer, write once
  def format_rows(rows, buf):
      buf.reset()
      for r in rows: buf.write(r.id); buf.write("-"); buf.write(r.name); buf.write("\n")
      return buf.view()   # no per-row allocation
  ```

## Streaming vs buffering

- **[MUST]** **Stream** large payloads (exports, bulk imports, large result sets, file transforms)
  in **bounded batches** — never load the whole thing into memory. Memory use must be O(batch),
  not O(dataset).

  ```
  # Buffering: O(N) memory, OOM risk on big tenants
  rows = db.fetch_all(query); csv = render_all(rows); return csv   # BAD

  # Streaming: O(batch) memory, constant footprint
  with db.cursor(query, fetch_size = 1000) as cur, out.stream() as sink:
      for batch in cur.batches():
          sink.write(render(batch))     # backpressure-aware; flush per batch
  ```

- **[MUST]** Use **server-side cursors / chunked responses**; set a fetch size so the driver
  doesn't buffer the full result. Apply backpressure so a slow consumer throttles the producer.

## Lazy evaluation & doing less

- **[SHOULD]** Compute lazily — don't materialize data a request won't use. Defer expensive derived
  fields until requested; short-circuit boolean/validation chains cheapest-check-first.
- **[SHOULD]** Avoid **premature/over-serialization**: don't serialize→deserialize between internal
  layers, don't build a full DTO graph when the caller needs three fields, and don't
  double-encode (e.g. JSON string inside JSON). Serialize once, at the boundary.

## Memoization & caching tiers (LRU + TTL)

- **[MUST]** Every cache is **bounded and evicting** — `max_size` + `TTL` (+ LRU/LFU eviction).
  An unbounded cache is a memory leak (Standard 03).
- **[SHOULD]** Use tiered caching, cheapest/nearest first:

  ```
  L1  in-process (LRU+TTL)     ~ns–µs   per-instance, small, hottest keys
  L2  shared cache (Redis/…)   ~ms      cross-instance, larger, tenant-scoped keys
  L3  origin (DB/service)      ~10ms+   source of truth
  ```

- **[MUST]** **Tenant-scope every cache key** (`{tenant}:{entity}:{id}`) — never let one tenant read
  another's cached data.
- **[MUST]** Have an **invalidation strategy**: write-through, or event-based invalidation, or a
  short TTL with staleness you can tolerate. Prefer short TTL + explicit bust on write for
  correctness-sensitive data.
- **[SHOULD]** Guard against **stampede/thundering herd** on hot-key expiry: single-flight
  (coalesce concurrent misses into one origin call), early/probabilistic refresh, or a brief
  serve-stale-while-revalidate window.
- **[SHOULD]** **Memoize** pure, expensive, deterministic computations per-request (or globally with
  a bounded cache) keyed on inputs.

## Batch I/O

- **[MUST]** Batch chatty I/O: one round trip of 500 items beats 500 round trips (bulk insert,
  multi-get, pipelined commands). Coalesce writes with a bounded buffer flushed by size **or** time
  (`flush when count >= 500 or age >= 50 ms`).
- **[SHOULD]** Debounce/coalesce duplicate in-flight requests (single-flight) for identical keys.

## Async for slow work

- **[MUST]** Anything > ~1 s or third-party runs as a **retried, idempotent background job with a
  DLQ** (Standard 03) — the request returns a job id; the UI polls/subscribes. Applies to PDFs,
  messaging, external pushes, imports/exports.
- **[MUST]** Jobs are **idempotent** (safe to retry) and carry a dedupe key. Bound worker
  concurrency and apply per-tenant fairness (see [01 · Pooling](01-pooling-and-resources.md)).
- **[SHOULD]** Use non-blocking async I/O for fan-out (issue N independent calls concurrently with a
  bounded concurrency limit) rather than sequential awaits.

  ```
  # Sequential: latency = sum of all calls
  for id in ids: results.append(fetch(id))            # BAD for independent calls

  # Bounded-concurrency fan-out: latency ~ slowest call
  results = gather(fetch(id) for id in ids, limit = 8)  # cap concurrency to protect upstream/pool
  ```

## Acceptance checklist

- [ ] Hot paths identified by profiler; optimization ranked by frequency × cost, not by guess.
- [ ] Hot-path allocations minimized; GC pause p99 / allocation rate monitored.
- [ ] Large payloads streamed in bounded batches with server-side cursors and backpressure.
- [ ] No premature/double serialization between internal layers; serialize once at the boundary.
- [ ] All caches bounded (max_size + TTL + eviction), tenant-scoped, with a defined invalidation.
- [ ] Stampede protection (single-flight / stale-while-revalidate) on hot keys.
- [ ] Chatty I/O batched; writes coalesced by size-or-time; duplicate in-flight calls single-flighted.
- [ ] Slow/third-party work is async, idempotent, DLQ-backed; fan-out uses bounded concurrency.
