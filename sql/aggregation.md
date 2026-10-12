# Aggregation: GROUP BY, HAVING and FILTER

> Turn many rows into summary rows — counts, sums, averages — per group.

[← Handbook](../README.md) · [All SQL guides](README.md)

---

## The order SQL actually runs in

```
FROM / JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

That's why you can't use a `SELECT` alias in `WHERE`, and why `WHERE` filters rows while `HAVING`
filters groups.

## Basics

```sql
SELECT team_id,
       count(*)                       AS tasks,
       count(completed_at)            AS done,          -- count(col) skips NULLs
       avg(completed_at - created_at) AS avg_time_to_done,
       max(created_at)                AS latest
FROM tasks
WHERE created_at >= now() - interval '30 days'          -- filter rows
GROUP BY team_id
HAVING count(*) >= 5                                     -- filter groups
ORDER BY tasks DESC;
```

Every non-aggregated column in `SELECT` must be in `GROUP BY` (or functionally dependent on the
grouped primary key).

## `FILTER` — several conditional aggregates in one pass

```sql
SELECT team_id,
       count(*)                                    AS total,
       count(*) FILTER (WHERE status = 'open')     AS open,
       count(*) FILTER (WHERE status = 'done')     AS done,
       round(100.0 * count(*) FILTER (WHERE status = 'done') / nullif(count(*), 0), 1) AS pct_done
FROM tasks
GROUP BY team_id;
```

Much clearer than `sum(CASE WHEN … THEN 1 ELSE 0 END)`, and one table scan instead of several queries.

## Percentages and division

- Use `100.0 *` (a numeric) or cast — `done / total` with integers does integer division.
- `nullif(total, 0)` avoids division by zero and yields `NULL` ("no data") instead of an error.
- `avg()` over no rows returns `NULL`, not `0` — keep it that way in APIs; "no data" isn't "zero".

## Useful aggregates

| Function | Use |
|---|---|
| `count(DISTINCT user_id)` | Unique values |
| `string_agg(name, ', ' ORDER BY name)` | Concatenate into a string |
| `array_agg(id ORDER BY created_at)` | Collect into an array |
| `jsonb_agg(row_to_json(t))` / `jsonb_object_agg(k, v)` | Build JSON in SQL |
| `percentile_cont(0.95) WITHIN GROUP (ORDER BY latency_ms)` | p95 |
| `bool_or(flag)`, `bool_and(flag)` | Any / all |

## GROUPING SETS and ROLLUP — subtotals in one query

```sql
SELECT team_id, status, count(*)
FROM tasks
GROUP BY ROLLUP (team_id, status);   -- per team+status, per team, and a grand total
```

## Pitfalls

- Joining one-to-many *before* aggregating multiplies rows (counts and sums come out too big).
  Aggregate each side in a subquery, then join.
- `count(*)` on huge filtered sets for every API request is expensive — cache, approximate, or drop totals.

## Related

[Window functions](window-functions.md) · [Joins](joins.md) · [NULLs and gotchas](nulls-and-gotchas.md)

## Further reading

- PostgreSQL docs, *Aggregate Functions*: https://www.postgresql.org/docs/current/functions-aggregate.html
