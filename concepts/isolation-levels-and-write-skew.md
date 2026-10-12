# Isolation levels and write skew

> Isolation decides **how much concurrent transactions can see of each other** — and which
> anomalies you must handle yourself.

[← Handbook](../README.md) · [All concepts](README.md)

---

## The anomalies

| Anomaly | What happens |
|---|---|
| **Dirty read** | Reading another transaction's uncommitted changes |
| **Non-repeatable read** | Reading the same row twice in one transaction and getting different values |
| **Phantom** | Re-running a query and getting new rows that match |
| **Lost update** | Two read-modify-write cycles; the second overwrites the first |
| **Write skew** | Two transactions read overlapping data, write *different* rows, and together break a rule |

## PostgreSQL's levels

| Level | Snapshot | Prevents | Still possible |
|---|---|---|---|
| **Read Committed** (default) | New snapshot per statement | Dirty reads | Non-repeatable reads, phantoms, lost updates (via read-modify-write), write skew |
| **Repeatable Read** | One snapshot for the whole transaction | + non-repeatable reads, phantoms; concurrent updates to the *same row* fail with a serialization error | **Write skew** |
| **Serializable** | Snapshot + conflict detection (SSI) | Everything — the result equals *some* serial order | Nothing, but transactions may fail with `40001` and must be **retried** |

Postgres treats `READ UNCOMMITTED` as Read Committed, so dirty reads never happen.

## Write skew, step by step

Rule: *at least one doctor must be on call.* Alice and Bob are both on call.

| Time | Transaction A (Alice) | Transaction B (Bob) |
|---|---|---|
| 1 | `SELECT count(*) … on_call` → 2 | |
| 2 | | `SELECT count(*) … on_call` → 2 |
| 3 | `UPDATE … SET on_call = false WHERE name = 'Alice'` | |
| 4 | | `UPDATE … SET on_call = false WHERE name = 'Bob'` |
| 5 | `COMMIT` | `COMMIT` |

Each saw two doctors and removed only themselves. Both commit. **Nobody is on call.** They wrote
different rows, so row locks never collided — Read Committed *and* Repeatable Read allow this.

## Fixes

1. **Serializable + retry loop** — Postgres detects the dangerous pattern and aborts one transaction.
   ```ts
   async function withSerializableRetry<T>(fn: () => Promise<T>, attempts = 3): Promise<T> {
     for (let i = 1; ; i++) {
       try { return await db.transaction(fn, { isolationLevel: 'Serializable' }) }
       catch (e) { if (!isSerializationFailure(e) || i === attempts) throw e; await sleep(jitter(i)) }
     }
   }
   ```
2. **Lock a shared row first**, so the two transactions serialize:
   ```sql
   SELECT id FROM shifts WHERE id = $1 FOR UPDATE;   -- both transactions lock the same parent row
   ```
3. **Encode the rule as a constraint** where possible (unique, check, exclusion constraints).

## Fixing lost updates (simpler cousin)

- Atomic statements: `UPDATE accounts SET balance = balance - 30 WHERE id = $1`.
- [Optimistic concurrency](optimistic-concurrency-etags.md) with a `version` column.
- `SELECT … FOR UPDATE` before modifying.

## Pitfalls

- Assuming "transactions make it safe" while running at Read Committed.
- Using Serializable without a retry loop — it *will* abort transactions under contention.
- Retrying non-idempotent side effects (emails) inside the retried block.

## Related

[MVCC and VACUUM](mvcc-and-vacuum.md) · [Locking and SKIP LOCKED](../sql/locking-and-skip-locked.md) ·
[Optimistic concurrency](optimistic-concurrency-etags.md)

## Further reading

- PostgreSQL docs, *Transaction Isolation*: https://www.postgresql.org/docs/current/transaction-iso.html
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapter 7 ("Transactions")
