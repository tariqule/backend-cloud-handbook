# SQL guides (PostgreSQL)

[← Back to the handbook](../README.md) · [Concept deep dives](../concepts/README.md)

Practical SQL for backend engineers, with runnable examples on the handbook's TaskFlow schema
(`teams`, `tasks`, `users`). Work through them in order, or jump to what you need.

| Guide | You'll learn |
|---|---|
| [Joins](joins.md) | Inner, outer, anti- and semi-joins, the `ON` vs `WHERE` trap, `LATERAL` |
| [Aggregation](aggregation.md) | `GROUP BY`, `HAVING`, `FILTER`, percentages without division bugs, `ROLLUP` |
| [CTEs and subqueries](ctes-and-subqueries.md) | Readable multi-step queries, data-modifying CTEs, recursive CTEs |
| [Window functions](window-functions.md) | Latest per group, ranking, running totals, `lag`/`lead`, `DISTINCT ON` |
| [Upserts and RETURNING](upserts-and-returning.md) | `INSERT … ON CONFLICT`, conditional upserts, counters, `MERGE` |
| [Reading EXPLAIN ANALYZE](explain-analyze.md) | Plan nodes, estimates vs actuals, buffers, `pg_stat_statements` |
| [Locks and SKIP LOCKED](locking-and-skip-locked.md) | Row locks, advisory locks, a Postgres job queue, avoiding lock outages |
| [NULLs and gotchas](nulls-and-gotchas.md) | `NULL` logic, `NOT IN`, row comparisons, time zones, integer division |

## A practice schema

```sql
CREATE TABLE teams (
  id         bigserial PRIMARY KEY,
  org_id     uuid NOT NULL,
  name       text NOT NULL,
  deleted_at timestamptz
);
CREATE UNIQUE INDEX teams_name_live ON teams (org_id, name) WHERE deleted_at IS NULL;

CREATE TABLE users (
  id    bigserial PRIMARY KEY,
  email text NOT NULL UNIQUE,
  name  text NOT NULL
);

CREATE TABLE tasks (
  id           bigserial PRIMARY KEY,
  team_id      bigint REFERENCES teams (id),
  assignee_id  bigint REFERENCES users (id),
  parent_id    bigint REFERENCES tasks (id),
  title        text NOT NULL,
  status       text NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'done', 'archived')),
  version      int  NOT NULL DEFAULT 1,
  created_at   timestamptz NOT NULL DEFAULT now(),
  completed_at timestamptz
);
CREATE INDEX tasks_team_created ON tasks (team_id, created_at DESC, id DESC);
CREATE INDEX tasks_assignee ON tasks (assignee_id);

-- Some data to play with
INSERT INTO teams (org_id, name)
SELECT gen_random_uuid(), 'Team ' || g FROM generate_series(1, 20) g;

INSERT INTO users (email, name)
SELECT 'user' || g || '@example.com', 'User ' || g FROM generate_series(1, 200) g;

INSERT INTO tasks (team_id, assignee_id, title, status, created_at, completed_at)
SELECT (random() * 19 + 1)::int,
       (random() * 199 + 1)::int,
       'Task ' || g,
       CASE WHEN random() < 0.6 THEN 'done' ELSE 'open' END,
       now() - (random() * interval '90 days'),
       NULL
FROM generate_series(1, 200000) g;

UPDATE tasks SET completed_at = created_at + (random() * interval '10 days') WHERE status = 'done';
ANALYZE;
```

Run it in a local Postgres (`docker run -e POSTGRES_PASSWORD=pw -p 5432:5432 postgres:18`), then try
every example in the guides and read its plan with `EXPLAIN (ANALYZE, BUFFERS)`.
