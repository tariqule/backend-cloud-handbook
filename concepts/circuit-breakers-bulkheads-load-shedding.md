# Circuit breakers, bulkheads and load shedding

> Three patterns that stop **one failing dependency — or too much traffic — from taking down the
> whole service**.

[← Handbook](../README.md) · [All concepts](README.md)

---

## Timeouts come first

None of these work without a **timeout on every network call**. A call with no timeout can hold a
connection, a request slot and memory forever. Set timeouts from the dependency's normal latency
(e.g. a little above its p99) and from your own deadline.

## Circuit breaker

Stop calling a dependency that keeps failing, and fail fast instead.

```
CLOSED ── failure rate above threshold ──► OPEN ── after a cool-down ──► HALF-OPEN
  ▲                                         │                              │
  └───────── trial requests succeed ────────┴──── trial requests fail ─────┘ (back to OPEN)
```

- **Closed:** calls go through; failures are counted.
- **Open:** calls fail immediately (return a fallback or an error) — no waiting on timeouts.
- **Half-open:** a few trial calls test whether the dependency has recovered.

Benefits: callers get fast answers, and the struggling dependency gets breathing room.

## Bulkhead

Isolate resources per dependency so one can't exhaust everything — like watertight compartments in
a ship.
- Separate connection pools or concurrency limits per downstream service.
- A slow calendar API can then use up only *its* 20 slots, not all 200 the service has.

```ts
const calendarLimit = pLimit(20)        // max 20 concurrent calendar calls
const getEvents = (id: string) => calendarLimit(() => calendar.events(id, { signal: AbortSignal.timeout(1500) }))
```

## Load shedding

When the service itself is overloaded, **reject some work early** rather than letting every request
slow down.
- Return `503` with `Retry-After` when queues, concurrency or latency pass a threshold.
- Shed low-priority work first (background refreshes, analytics) and protect critical paths
  (checkout, login).
- Rate-limit per client (token bucket) so one noisy caller can't starve the rest.

## Graceful degradation

Return partial results when a non-critical dependency fails: the task page loads without the
"related calendar events" panel instead of failing entirely.

## Pitfalls

- Breakers without timeouts — they never see the failure until it's too late.
- One shared pool for every dependency (no bulkhead).
- Thresholds set once and never revisited as traffic grows.

## Related

[Retries, backoff and jitter](retries-backoff-jitter.md) · [Little's Law](littles-law.md) ·
[SLOs, error budgets and burn rate](slos-error-budgets-burn-rate.md)

## Further reading

- Martin Fowler, *Circuit Breaker*: https://martinfowler.com/bliki/CircuitBreaker.html
- Michael Nygard, *Release It!* (stability patterns)
