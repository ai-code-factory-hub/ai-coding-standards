# 01 · Pooling & Resources

**Purpose:** size and govern every finite resource — DB connections, threads/workers, HTTP
sockets, memory — so latency stays flat under load and one tenant can never starve another.
Implements the pooling rules of [../standards-kb/03-performance-scalability.md](../standards-kb/03-performance-scalability.md).

## Connection pooling (databases)

- **[MUST]** Open connections through a **bounded pool**, never per-request. Every pool sets:
  `max_size`, `min_idle`, **`acquire_timeout`** (fail fast when saturated), **`max_lifetime`**
  (recycle to survive DB failovers / DNS changes), and `idle_timeout`.
- **[MUST]** Set an **`acquire_timeout` (≈ 1–3 s)**, not infinite. A blocked acquire that never
  times out turns a DB hiccup into an app-wide hang. On timeout, shed load (503 + Retry-After).
- **[MUST]** `max_lifetime` (≈ 15–30 min) so no connection outlives a DB restart or credential
  rotation. Add small jitter so the whole pool doesn't recycle at once.
- **[SHOULD]** Size the pool from a formula, then verify by load test. A common starting point:

  ```
  # Little's Law form: connections needed = throughput * avg_hold_time
  pool_size = ceil(peak_qps * avg_query_seconds) + headroom
  # e.g. 400 qps * 0.010 s hold = 4 busy, add headroom -> ~8-10

  # Upper bound is set by the DB, not the app:
  sum(all_app_instances * pool_size) + admin_reserve  <=  db_max_connections
  ```

  Smaller is usually faster: an oversized pool creates DB-side context-switch and lock
  contention. Prefer many short holds over a few long ones.
- **[MUST]** **Front the DB with a pooler** (e.g. a transaction-mode proxy) when you have many
  app instances or serverless/autoscaled workers — thousands of app-side connections collapse to
  a small server-side set.
  > **Example (PgBouncer / RDS Proxy):** app pools stay tiny (`max=5` each) and talk to PgBouncer
  > in `transaction` mode; PgBouncer keeps a bounded server pool of ~40 to Postgres. Note:
  > transaction-mode poolers forbid session-level state (session `SET`, server-side prepared
  > statements) — use protocol-level prepared statements or plan-cache-friendly SQL instead.

## Thread / worker pools

- **[MUST]** Bound every executor. An unbounded thread pool + unbounded queue is a memory leak
  and a latency amplifier. Size by workload type:

  ```
  cpu_bound_pool   = num_cores            # +1 spare at most
  io_bound_pool    = num_cores * (1 + wait_time/service_time)   # Brainerd/Little's ratio
  ```

- **[MUST]** Give the pool a **bounded queue with a rejection policy** (reject → 503, or
  caller-runs backpressure). Never `queue = unbounded` — it hides overload until OOM.
- **[MUST]** **Separate pools per workload class** so a slow class can't starve a fast one
  (see bulkheads below). CPU-bound rendering, blocking I/O, and latency-critical request handling
  each get their own executor.
- **[SHOULD]** In event-loop / async runtimes, keep the loop non-blocking; offload any CPU or
  blocking-I/O work to a **dedicated worker pool**, never run it on the loop thread.

## HTTP client pools (outbound)

- **[MUST]** Reuse a **keep-alive connection pool per upstream host**; never create a new client
  per call (TLS handshake + slow-start on every request destroys tail latency).
- **[MUST]** Set **per-route max connections**, **connection + read timeouts**, and a **pool
  acquire timeout**. Pair with the circuit breakers/retries in
  [../standards-kb/05-reliability-resilience.md](../standards-kb/05-reliability-resilience.md).
- **[SHOULD]** Cap total sockets so a slow upstream can't consume every ephemeral port. Enable
  HTTP/2 or connection multiplexing where the upstream supports it.

## Resource limits (memory, files, descriptors)

- **[MUST]** Set an explicit **process/container memory limit** and a heap/GC ceiling below it,
  so the runtime GCs before the OOM-killer strikes.
- **[MUST]** Deterministically **close every acquired resource** (connections, file handles,
  streams) with try-with-resources / `defer` / context managers — leaked handles exhaust the pool.
- **[SHOULD]** Cap in-flight request body size and concurrent uploads; raise the file-descriptor
  ulimit to match `pool_size + sockets + headroom`.

## Avoiding pool exhaustion

- **[MUST]** **Never hold a pooled connection across a network/third-party call.** Acquire late,
  release early. The classic exhaustion bug: `acquire db → call external API (2 s) → release`.
- **[MUST]** Emit pool metrics: `active`, `idle`, `waiting`, `acquire_wait_ms` p95,
  `acquire_timeouts`. Alert when `waiting > 0` sustained or acquire p95 approaches the timeout.
- **[SHOULD]** Detect leaks: log a stack trace when a connection is held longer than a threshold
  (e.g. `leak_detection_threshold = 5 s`).

## Per-tenant fairness & bulkheads

- **[MUST]** In a shared pool, a single hot tenant can consume all slots. Add a **per-tenant
  concurrency cap (bulkhead)** so no tenant exceeds its share:

  ```
  # semaphore per tenant, sized as a share of the shared pool
  per_tenant_limit = max(1, floor(pool_size * fairness_fraction))   # e.g. 20% cap

  handle(request):
      sem = tenant_semaphores[request.tenant_id]   # bounded per tenant
      if not sem.try_acquire(timeout = 50ms):
          return 429  # this tenant is at its cap; others unaffected
      try:
          conn = pool.acquire(timeout = 2s)
          ... do work ...
      finally:
          release(conn); sem.release()
  ```

- **[SHOULD]** For strong isolation of large/enterprise tenants, give them a **dedicated pool or
  worker set** (partitioned bulkhead) rather than a share of the common one.
- **[SHOULD]** Apply the same fairness at the job queue: weighted/round-robin dequeue per tenant
  so a bulk importer can't monopolize workers. See async queues in
  [03 · Code-Level Performance](03-code-level-performance.md).

> **Example (multi-tenant LIS):** nightly bulk result-import for one large lab would drain the
> shared DB pool and stall interactive registration for every other tenant. A 20% per-tenant
> semaphore caps the importer; small tenants keep sub-100 ms acquires while the big job simply
> runs at bounded concurrency.

## Acceptance checklist

- [ ] Every pool sets `max_size`, `acquire_timeout`, `max_lifetime`, `idle_timeout`.
- [ ] DB pool size derived from a formula and validated; `sum(instances*pool) <= db_max_connections`.
- [ ] DB fronted by a pooler where instance count / autoscaling warrants it.
- [ ] Thread/worker pools bounded, with a bounded queue and a rejection (backpressure) policy.
- [ ] HTTP clients use per-host keep-alive pools with connect/read/acquire timeouts.
- [ ] Memory limit + GC ceiling set; all resources closed deterministically.
- [ ] No pooled connection is held across a third-party call; pool metrics exported and alerted.
- [ ] Per-tenant bulkhead/fairness prevents one tenant from exhausting shared pools.
