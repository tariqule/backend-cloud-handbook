# Row locks, advisory locks and SKIP LOCKED

> Lock exactly the rows you're about to change, for as short a time as possible — and use
> `SKIP LOCKED` to build concurrency-safe job queues in plain Postgres.

[← Handbook](../README.md) · [All SQL guides](README.md)

---

## Row-level locks

```sql
BEGIN;
SELECT * FROM accounts WHERE id = $1 FOR UPDATE;      -- others wanting to change this row wait
UPDATE accounts SET balance = balance - $2 WHERE id = $1;
COMMIT;                                                -- lock released
```

| Clause | Blocks | Use when |
|---|---|---|
| `FOR UPDATE` | Other updates, deletes and `FOR UPDATE`/`FOR SHARE` | You'll update or delete the row, including its key |
| `FOR NO KEY UPDATE` | Same, but lets others take `FOR KEY SHARE` | You'll update non-key columns (less contention with foreign keys) |
| `FOR SHARE` | Updates and deletes | You need the row not to change, without changing it |
| `FOR KEY SHARE` | Deletes and key changes | Rarely used directly (foreign keys use it) |

Plain `SELECT`s never block on these locks — MVCC lets readers read.

## Options

```sql
SELECT … FOR UPDATE NOWAIT;      -- error immediately if locked
SELECT … FOR UPDATE SKIP LOCKED; -- skip locked rows and return the rest
```

## SKIP LOCKED: a job queue in Postgres

```sql
CREATE TABLE jobs (
  id         bigserial PRIMARY KEY,
  kind       text NOT NULL,
  payload    jsonb NOT NULL,
  status     text NOT NULL DEFAULT 'queued',     -- queued | running | done | failed
  attempts   int  NOT NULL DEFAULT 0,
  run_at     timestamptz NOT NULL DEFAULT now(),
  locked_at  timestamptz
);
CREATE INDEX jobs_ready ON jobs (run_at) WHERE status = 'queued';

-- Each worker claims up to 10 jobs; workers never block each other or claim the same job
WITH next AS (
  SELECT id FROM jobs
  WHERE status = 'queued' AND run_at <= now()
  ORDER BY run_at
  FOR UPDATE SKIP LOCKED
  LIMIT 10
)
UPDATE jobs j SET status = 'running', locked_at = now(), attempts = attempts + 1
FROM next WHERE j.id = next.id
RETURNING j.*;
```

Then mark each job `done`, or set it back to `queued` with a later `run_at` (backoff) on failure,
or `failed` after too many attempts. A sweeper re-queues jobs stuck in `running` past a timeout
(the worker crashed).

Good up to a moderate volume; beyond that, a dedicated queue (Pub/Sub, SQS) is simpler to operate.

## Advisory locks: application-defined mutexes

```sql
-- "Only one instance runs the nightly report at a time"
SELECT pg_try_advisory_xact_lock(hashtext('nightly-report'));   -- true = you got it; released at COMMIT
```

Use the **transaction-scoped** (`_xact_`) versions behind a transaction-mode pooler; session-level
advisory locks can be held by a pooled connection after your client has moved on.

## Avoiding lock trouble

- **Lock in a consistent order** (e.g. by id) to avoid deadlocks; Postgres aborts one side with
  `40P01` — retry it.
- **Keep transactions short;** never hold locks across HTTP calls or user think-time.
- **DDL needs strong locks.** An `ALTER TABLE` waiting behind a long transaction blocks every new
  query on that table. Always `SET lock_timeout = '3s'` in migrations.
- Inspect blocking:
  ```sql
  SELECT pid, pg_blocking_pids(pid) AS blocked_by, state, left(query, 60)
  FROM pg_stat_activity
  WHERE cardinality(pg_blocking_pids(pid)) > 0;
  ```

## Related

[Isolation levels and write skew](../concepts/isolation-levels-and-write-skew.md) ·
[Outbox pattern](../concepts/outbox-pattern.md) · [Zero-downtime migrations](../concepts/zero-downtime-migrations.md)

## Further reading

- PostgreSQL docs, *Explicit Locking*: https://www.postgresql.org/docs/current/explicit-locking.html
- PostgreSQL docs, `SELECT` — *The Locking Clause*: https://www.postgresql.org/docs/current/sql-select.html
