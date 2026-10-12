# Window functions

> Compute values **across related rows without collapsing them** — ranks, running totals,
> "latest per group", comparisons with the previous row.

[← Handbook](../README.md) · [All SQL guides](README.md)

---

## The shape

```sql
function(...) OVER (PARTITION BY <group> ORDER BY <order> [frame])
```

- `PARTITION BY` — restart the calculation for each group (like `GROUP BY`, but rows stay).
- `ORDER BY` — order within the partition.
- Frame (`ROWS BETWEEN …`) — which rows around the current one to include.

## Latest row per group (the most common use)

```sql
-- Each user's most recent task
SELECT * FROM (
  SELECT k.*, row_number() OVER (PARTITION BY assignee_id ORDER BY created_at DESC, id DESC) AS rn
  FROM tasks k
) x
WHERE rn = 1;

-- Postgres shortcut for the same thing
SELECT DISTINCT ON (assignee_id) *
FROM tasks
ORDER BY assignee_id, created_at DESC, id DESC;
```

`DISTINCT ON` is the idiomatic way to compute "current state = latest event" in
[append-only event tables](../README.md#chapter-8--schema-evolution-and-data-modelling).

## Ranking

```sql
SELECT assignee_id, done,
       row_number() OVER w AS row_num,   -- 1,2,3,4 (no ties)
       rank()       OVER w AS rnk,       -- 1,2,2,4 (gaps after ties)
       dense_rank() OVER w AS dense      -- 1,2,2,3 (no gaps)
FROM (SELECT assignee_id, count(*) AS done FROM tasks WHERE status = 'done' GROUP BY 1) s
WINDOW w AS (ORDER BY done DESC);
```

## Running totals and moving averages

```sql
SELECT day, tasks_done,
       sum(tasks_done) OVER (ORDER BY day)                                        AS cumulative,
       avg(tasks_done) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS avg_7d
FROM daily_stats;
```

## Comparing with neighbours

```sql
SELECT day, tasks_done,
       tasks_done - lag(tasks_done) OVER (ORDER BY day) AS change_vs_yesterday,
       lead(day) OVER (ORDER BY day)                     AS next_day
FROM daily_stats;
```

## Share of total

```sql
SELECT team_id, count(*) AS tasks,
       round(100.0 * count(*) / sum(count(*)) OVER (), 1) AS pct_of_all
FROM tasks
GROUP BY team_id;
```

## Notes

- Window functions run **after** `WHERE`, `GROUP BY` and `HAVING`, so you can't filter on them
  directly — wrap the query (as above) and filter outside.
- Add a unique tie-breaker (`id`) to `ORDER BY` for deterministic results.
- An index matching `(PARTITION BY cols, ORDER BY cols)` can avoid a large sort.

## Related

[Aggregation](aggregation.md) · [CTEs and subqueries](ctes-and-subqueries.md) ·
[Keyset pagination](../concepts/keyset-pagination.md)

## Further reading

- PostgreSQL docs, *Window Functions* tutorial: https://www.postgresql.org/docs/current/tutorial-window.html
