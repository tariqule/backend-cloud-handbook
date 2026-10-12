# Concept deep dives

[← Back to the handbook](../README.md) · [SQL guides](../sql/README.md)

Short, focused pages on the key terms used in the handbook. Each explains the idea, shows code,
lists the pitfalls and links to authoritative sources.

## APIs and services

| Concept | One line |
|---|---|
| [Idempotency](idempotency.md) | Make retries harmless with idempotency keys and idempotent consumers |
| [Optimistic concurrency and ETags](optimistic-concurrency-etags.md) | Prevent lost updates with versions, `If-Match` and `412` |
| [Keyset pagination](keyset-pagination.md) | Fast, stable paging with cursors and tie-breakers |
| [Event loop and libuv threadpool](event-loop-and-libuv.md) | What blocks Node, and the 4-thread pool most people forget |
| [Streams and backpressure](streams-and-backpressure.md) | Process large data without blowing up memory |

## Data

| Concept | One line |
|---|---|
| [MVCC and VACUUM](mvcc-and-vacuum.md) | How Postgres versions rows, and why long transactions hurt |
| [Isolation levels and write skew](isolation-levels-and-write-skew.md) | Which anomalies each level allows, and how to prevent them |
| [Indexing](indexing.md) | Index types, composite order, partial/covering/expression indexes |
| [The N+1 query problem](n-plus-one.md) | The most common performance bug, and how to batch it away |
| [Connection pooling](connection-pooling.md) | Connection budgets, PgBouncer modes, RDS Proxy |
| [Zero-downtime migrations](zero-downtime-migrations.md) | Expand → migrate → contract, lock timeouts, safe recipes |
| [Caching strategies](caching-strategies.md) | Cache-aside, invalidation after commit, stampede protection |

## Distributed systems and reliability

| Concept | One line |
|---|---|
| [Outbox pattern](outbox-pattern.md) | Publish events reliably without dual writes |
| [Sagas](sagas.md) | Multi-step workflows with compensating actions |
| [Retries, backoff and jitter](retries-backoff-jitter.md) | Safe retries, retry budgets, avoiding amplification |
| [Circuit breakers, bulkheads and load shedding](circuit-breakers-bulkheads-load-shedding.md) | Contain failures and overload |
| [Little's Law](littles-law.md) | Concurrency = throughput × latency, and why slow dependencies cascade |
| [CAP and PACELC](cap-and-pacelc.md) | Consistency vs availability vs latency, per operation |
| [SLOs, error budgets and burn rate](slos-error-budgets-burn-rate.md) | Measure reliability and alert on what matters |
| [Observability signals](observability-signals.md) | Logs, metrics, traces, RED/USE and cardinality |

## Security and cloud

| Concept | One line |
|---|---|
| [JWTs and token verification](jwt-and-token-verification.md) | Signature, issuer, audience, expiry, JWKS rotation, PKCE |
| [Authorization models](authorization-models.md) | RBAC, ABAC, ReBAC and structural authorization |
| [Row-level security](row-level-security.md) | Database-enforced tenant isolation in Postgres |
| [SSRF](ssrf.md) | Stop your server being used to reach internal targets |
| [Least privilege in cloud IAM](least-privilege-iam.md) | Scoping identities, and how GCP and AWS evaluate permissions |
| [Workload identity and keyless CI/CD](workload-identity-federation.md) | No more key files: OIDC federation on GCP and AWS |
