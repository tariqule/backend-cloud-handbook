# CTEs and subqueries

> Name intermediate results so complex queries read **top to bottom, step by step**.

[← Handbook](../README.md) · [All SQL guides](README.md)

---

## Subqueries

```sql
-- Scalar subquery (one value)
SELECT title, (SELECT name FROM teams WHERE id = tasks.team_id) AS team FROM tasks;

-- In WHERE
SELECT * FROM tasks WHERE team_id IN (SELECT id FROM teams WHERE org_id = $1);

-- Correlated (re-evaluated per outer row) — fine with an index, slow without one
SELECT * FROM teams t WHERE EXISTS (SELECT 1 FROM tasks k WHERE k.team_id = t.id);

-- Derived table (subquery in FROM)
SELECT team_id, avg(n) FROM (SELECT team_id, assignee_id, count(*) AS n FROM tasks GROUP BY 1, 2) x GROUP BY team_id;
```

## CTEs (`WITH`)

```sql
WITH recent AS (
  SELECT * FROM tasks WHERE created_at >= now() - interval '7 days'
),
per_team AS (
  SELECT team_id, count(*) AS created, count(completed_at) AS done
  FROM recent
  GROUP BY team_id
)
SELECT t.name, p.created, p.done
FROM per_team p
JOIN teams t ON t.id = p.team_id
ORDER BY p.created DESC;
```

Each step has a name, so the query reads like a pipeline. In PostgreSQL 12+, simple CTEs are
**inlined** into the main query and optimized together. Add `MATERIALIZED` to force a CTE to be
computed once (useful when it's referenced several times or to fence off a bad plan), or
`NOT MATERIALIZED` to force inlining.

## Data-modifying CTEs

Change data and use the result in the same statement:

```sql
WITH moved AS (
  UPDATE tasks SET team_id = $2 WHERE team_id = $1 AND status = 'open'
  RETURNING id
)
INSERT INTO audit_log (action, details)
SELECT 'task.moved', jsonb_build_object('task_id', id, 'to', $2) FROM moved;
```

One statement, one snapshot, atomic.

## Recursive CTEs — hierarchies and graphs

```sql
-- All sub-tasks under task $1, any depth
WITH RECURSIVE tree AS (
  SELECT id, parent_id, title, 1 AS depth FROM tasks WHERE id = $1
  UNION ALL
  SELECT k.id, k.parent_id, k.title, tree.depth + 1
  FROM tasks k JOIN tree ON k.parent_id = tree.id
  WHERE tree.depth < 20                      -- guard against cycles / runaway depth
)
SELECT * FROM tree;
```

## When to use which

| Use | When |
|---|---|
| CTE | Multi-step logic, readability, reusing an intermediate result, data-modifying steps, recursion |
| Subquery | Small, one-off filters (`EXISTS`, `IN`) |
| Join | Combining tables row by row |
| Lateral join | "Top N per group" and per-row subqueries — see [Joins](joins.md) |

## Related

[Joins](joins.md) · [Window functions](window-functions.md) · [Upserts and RETURNING](upserts-and-returning.md)

## Further reading

- PostgreSQL docs, *WITH Queries (Common Table Expressions)*: https://www.postgresql.org/docs/current/queries-with.html
