# NULLs, row comparisons and other SQL gotchas

> The handful of SQL behaviours that cause the most subtle bugs.

[← Handbook](../README.md) · [All SQL guides](README.md)

---

## NULL means "unknown"

Any comparison with `NULL` is `NULL` (not true, not false), and `WHERE` keeps only true rows.

```sql
SELECT NULL = NULL;          -- NULL (not true!)
SELECT NULL <> 1;            -- NULL
WHERE assignee_id = NULL     -- ❌ matches nothing
WHERE assignee_id IS NULL    -- ✅
WHERE a IS DISTINCT FROM b   -- ✅ NULL-safe "not equal"
```

## `NOT IN` with NULLs returns nothing

```sql
SELECT * FROM teams WHERE id NOT IN (SELECT team_id FROM tasks);
-- If any tasks.team_id is NULL, this returns ZERO rows.
```
Use `NOT EXISTS` instead — it behaves as you expect.

## Aggregates and NULL

- `count(*)` counts rows; `count(col)` skips NULLs.
- `sum`, `avg`, `min`, `max` ignore NULLs, and return **NULL** for no rows.
- `COALESCE(sum(x), 0)` when "nothing" should really display as zero — but in APIs, `null` often
  correctly means "no data".

## Ordering puts NULLs last (ascending) by default

```sql
ORDER BY due_date ASC NULLS LAST    -- be explicit
ORDER BY due_date DESC NULLS LAST   -- DESC puts NULLs first unless told otherwise
```

## Row-value comparisons

```sql
WHERE (created_at, id) < ($1, $2)
```
compares lexicographically — `created_at < $1 OR (created_at = $1 AND id < $2)` — and can use an
index on `(created_at, id)`. This is the core of [keyset pagination](../concepts/keyset-pagination.md).

## Integer division

```sql
SELECT 2 / 3;        -- 0
SELECT 2.0 / 3;      -- 0.666…
SELECT 2::numeric / 3;
```

## Time zones

- Use `timestamptz` (stores an instant; displays in the session time zone).
- `timestamp` without time zone stores wall-clock time with no zone — ambiguous across regions.
- "Today" depends on a time zone: `(created_at AT TIME ZONE 'America/Toronto')::date`.
- Ranges are half-open: `created_at >= '2026-10-01' AND created_at < '2026-10-02'` — never
  `BETWEEN … AND '2026-10-01 23:59:59'`.

## Text matching

- `=` is case-sensitive; use `lower(email) = lower($1)` with an expression index, or the `citext` type.
- `LIKE '%foo%'` can't use a B-tree index; use a trigram index (`pg_trgm`) or full-text search.

## Functions on indexed columns

```sql
WHERE date(created_at) = '2026-10-01'      -- ❌ can't use an index on created_at
WHERE created_at >= '2026-10-01' AND created_at < '2026-10-02'   -- ✅
```

## `SELECT *` in application code

Breaks when columns are added or reordered, fetches more than needed, and prevents index-only
scans. Name the columns.

## Related

[Joins](joins.md) · [Aggregation](aggregation.md) · [Indexing](../concepts/indexing.md)

## Further reading

- PostgreSQL docs, *Comparison Functions and Operators*: https://www.postgresql.org/docs/current/functions-comparison.html
