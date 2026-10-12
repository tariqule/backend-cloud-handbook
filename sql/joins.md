# SQL joins

> Combine rows from two tables on a condition. The join type decides **what happens to rows with
> no match**.

[← Handbook](../README.md) · [All SQL guides](README.md)

---

## Sample tables

```sql
-- teams                     -- tasks
-- id | name                 -- id | team_id | title          | status
-- 1  | Design               -- 10 | 1       | Logo           | done
-- 2  | Backend              -- 11 | 1       | Colours        | open
-- 3  | Data                 -- 12 | 2       | API            | open
--                           -- 13 | NULL    | Unsorted idea  | open
```

## The join types

```sql
-- INNER JOIN: only matching pairs
SELECT t.name, k.title FROM teams t JOIN tasks k ON k.team_id = t.id;
-- Design/Logo, Design/Colours, Backend/API          (Data and "Unsorted idea" disappear)

-- LEFT JOIN: every left row, NULLs where no match
SELECT t.name, k.title FROM teams t LEFT JOIN tasks k ON k.team_id = t.id;
-- … plus Data/NULL

-- RIGHT JOIN: every right row (rarely used; swap the tables and use LEFT instead)

-- FULL OUTER JOIN: every row from both sides
SELECT t.name, k.title FROM teams t FULL JOIN tasks k ON k.team_id = t.id;
-- … plus Data/NULL and NULL/Unsorted idea

-- CROSS JOIN: every combination (use deliberately, e.g. with generate_series)
```

## Anti-join: "rows with no match"

```sql
-- Teams with no tasks
SELECT t.* FROM teams t
LEFT JOIN tasks k ON k.team_id = t.id
WHERE k.id IS NULL;

-- Same idea, often clearer
SELECT t.* FROM teams t
WHERE NOT EXISTS (SELECT 1 FROM tasks k WHERE k.team_id = t.id);
```

Avoid `WHERE id NOT IN (SELECT team_id FROM tasks)` — if the subquery returns a single `NULL`
(task 13), `NOT IN` returns **no rows at all**. See [NULLs and gotchas](nulls-and-gotchas.md).

## Semi-join: "rows that have at least one match"

```sql
SELECT t.* FROM teams t
WHERE EXISTS (SELECT 1 FROM tasks k WHERE k.team_id = t.id AND k.status = 'open');
```
`EXISTS` stops at the first match and never duplicates team rows, unlike a plain `JOIN`.

## `ON` vs `WHERE` with outer joins (the classic bug)

```sql
-- Intended: every team, with its OPEN tasks if any
SELECT t.name, k.title
FROM teams t
LEFT JOIN tasks k ON k.team_id = t.id AND k.status = 'open';   -- ✅ filter in ON

-- Bug: filtering the right table in WHERE turns it back into an inner join
SELECT t.name, k.title
FROM teams t
LEFT JOIN tasks k ON k.team_id = t.id
WHERE k.status = 'open';                                         -- ❌ teams without open tasks vanish
```

Conditions on the **optional** side of a `LEFT JOIN` belong in `ON`.

## LATERAL: "for each row, run this subquery"

```sql
-- Each team with its 3 most recent tasks
SELECT t.name, recent.title, recent.created_at
FROM teams t
CROSS JOIN LATERAL (
  SELECT title, created_at FROM tasks k
  WHERE k.team_id = t.id
  ORDER BY created_at DESC
  LIMIT 3
) recent;
```
Use `LEFT JOIN LATERAL (...) ON true` to keep teams with no tasks.

## Performance notes

- Index the join columns, especially foreign keys on the "many" side (`tasks.team_id`).
- Joining one-to-many then aggregating can multiply rows — aggregate in a subquery first if needed.

## Related

[Aggregation](aggregation.md) · [Window functions](window-functions.md) · [N+1 queries](../concepts/n-plus-one.md)

## Further reading

- PostgreSQL docs, *Joins Between Tables*: https://www.postgresql.org/docs/current/tutorial-join.html
