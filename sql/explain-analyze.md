# Reading EXPLAIN ANALYZE

> `EXPLAIN` shows the plan Postgres **intends** to use; `EXPLAIN ANALYZE` runs the query and shows
> what **actually** happened.

[← Handbook](../README.md) · [All SQL guides](README.md)

---

## Running it

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title FROM tasks
WHERE team_id = 42 AND status = 'open'
ORDER BY created_at DESC
LIMIT 50;
```

⚠️ `ANALYZE` **executes** the query. For `UPDATE`/`DELETE`, wrap it in `BEGIN; … ROLLBACK;`.

## Reading a plan

Plans are trees; read from the **innermost, most indented** node outwards. Each node shows:

```
Limit  (cost=0.43..12.10 rows=50 width=40) (actual time=0.05..0.31 rows=50 loops=1)
  Buffers: shared hit=54
  ->  Index Scan using tasks_open_by_team on tasks  (cost=… rows=1200 …) (actual … rows=50 loops=1)
        Index Cond: (team_id = 42)
Planning Time: 0.2 ms
Execution Time: 0.4 ms
```

| Part | Meaning |
|---|---|
| `cost=start..total` | The planner's estimate, in arbitrary units |
| `rows=` (first) | **Estimated** rows |
| `actual time=…` | Real milliseconds (start..end), **per loop** |
| `rows=` (actual) | **Real** rows, per loop |
| `loops=` | How many times the node ran — multiply time and rows by it |
| `Buffers: shared hit / read` | Pages found in memory vs read from disk |

## Node types to recognise

| Node | Means |
|---|---|
| **Seq Scan** | Reads the whole table. Fine for small tables; a red flag on big ones in hot queries |
| **Index Scan** | Walks an index, then fetches rows |
| **Index Only Scan** | Answered from the index alone (covering index, visibility map up to date) |
| **Bitmap Heap Scan** | Collects matching row locations from index(es), then reads pages in order |
| **Nested Loop** | For each outer row, look up inner rows — great with few outer rows and an index |
| **Hash Join** | Builds a hash table of one side — good for larger, unsorted joins |
| **Merge Join** | Joins two sorted inputs |
| **Sort** | Watch for `Sort Method: external merge Disk:` — it spilled to disk (raise `work_mem` or add an index) |

## What to look for

1. **Where the time goes** — the node whose actual time dominates.
2. **Estimate vs actual rows off by 10×–1000×** — the planner is guessing wrong. Run `ANALYZE`,
   raise the column's statistics target, or create extended statistics for correlated columns.
3. **Seq Scans on large tables** in frequent queries — a missing or unusable index.
4. **High `loops`** under a Nested Loop — often an N+1 inside one query.
5. **Disk reads (`read=`)** much higher than hits — cold cache or too much data touched.

## Finding *which* queries to analyse

```sql
-- Requires the pg_stat_statements extension
SELECT calls, round(total_exec_time) AS total_ms, round(mean_exec_time, 1) AS mean_ms, left(query, 80)
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Optimize by **total time** (frequency × cost), not just the single slowest query.

## Related

[Indexing](../concepts/indexing.md) · [N+1 queries](../concepts/n-plus-one.md) ·
[MVCC and VACUUM](../concepts/mvcc-and-vacuum.md)

## Further reading

- PostgreSQL docs, *Using EXPLAIN*: https://www.postgresql.org/docs/current/using-explain.html
- PostgreSQL docs, `pg_stat_statements`: https://www.postgresql.org/docs/current/pgstatstatements.html
