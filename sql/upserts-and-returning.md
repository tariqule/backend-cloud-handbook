# Upserts and RETURNING

> Insert-or-update in **one atomic statement**, and get the affected rows back without a second
> query.

[← Handbook](../README.md) · [All SQL guides](README.md)

---

## `INSERT … ON CONFLICT`

```sql
-- Insert, or update if the key already exists
INSERT INTO user_settings (user_id, theme, updated_at)
VALUES ($1, $2, now())
ON CONFLICT (user_id)
DO UPDATE SET theme = EXCLUDED.theme, updated_at = now();

-- Insert, or silently skip if it exists (idempotent inserts)
INSERT INTO processed_events (event_id) VALUES ($1)
ON CONFLICT (event_id) DO NOTHING;
```

- The conflict target must match a **unique index or constraint** (including a partial unique
  index, by repeating its predicate: `ON CONFLICT (org_id, name) WHERE deleted_at IS NULL`).
- `EXCLUDED` is the row you tried to insert.
- It's atomic and concurrency-safe — unlike "`SELECT`, then `INSERT` or `UPDATE`", which races.

## Conditional updates

```sql
-- Only overwrite with newer data (handles out-of-order events)
INSERT INTO task_status (task_id, status, version)
VALUES ($1, $2, $3)
ON CONFLICT (task_id) DO UPDATE
SET status = EXCLUDED.status, version = EXCLUDED.version
WHERE task_status.version < EXCLUDED.version;
```

## Counters

```sql
INSERT INTO daily_counts (day, team_id, n) VALUES (current_date, $1, 1)
ON CONFLICT (day, team_id) DO UPDATE SET n = daily_counts.n + 1;
```

## `RETURNING`

Get the rows back from `INSERT`, `UPDATE` or `DELETE`:

```sql
INSERT INTO tasks (team_id, title) VALUES ($1, $2) RETURNING id, created_at;

UPDATE tasks SET status = 'done', version = version + 1
WHERE id = $1 AND version = $2
RETURNING *;                       -- zero rows = version conflict (see optimistic concurrency)

DELETE FROM sessions WHERE expires_at < now() RETURNING user_id;
```

Saves a round trip and returns exactly what was written (defaults, generated ids, triggers' effects).

## `MERGE` (Postgres 15+)

For complex "match on a key, then insert / update / delete" logic, especially syncing from a
staging table:

```sql
MERGE INTO tasks t
USING staged_tasks s ON t.external_id = s.external_id
WHEN MATCHED AND s.deleted THEN DELETE
WHEN MATCHED THEN UPDATE SET title = s.title
WHEN NOT MATCHED THEN INSERT (external_id, title) VALUES (s.external_id, s.title);
```

For single-row upserts, `ON CONFLICT` remains simpler and fully concurrency-safe.

## Related

[Idempotency](../concepts/idempotency.md) · [Optimistic concurrency](../concepts/optimistic-concurrency-etags.md) ·
[CTEs and subqueries](ctes-and-subqueries.md)

## Further reading

- PostgreSQL docs, `INSERT` (including `ON CONFLICT`): https://www.postgresql.org/docs/current/sql-insert.html
- PostgreSQL docs, `MERGE`: https://www.postgresql.org/docs/current/sql-merge.html
