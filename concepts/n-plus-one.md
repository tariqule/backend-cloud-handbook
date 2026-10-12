# The N+1 query problem

> Running **one query to get a list, then one more query per item** — instead of one or two
> batched queries.

[← Handbook](../README.md) · [All concepts](README.md)

---

## What it looks like

```ts
const teams = await db.team.findMany({ where: { orgId } })          // 1 query
for (const team of teams) {
  team.openTasks = await db.task.count({ where: { teamId: team.id, status: 'open' } })  // N queries
}
```

50 teams → 51 round trips. Each is fast on its own, so it passes code review and local testing
(where N is 3). In production N is 500 and latency is dominated by round trips.

## Fixes

**Aggregate in one query**
```sql
SELECT t.id, t.name, count(k.id) FILTER (WHERE k.status = 'open') AS open_tasks
FROM teams t
LEFT JOIN tasks k ON k.team_id = t.id
WHERE t.org_id = $1
GROUP BY t.id, t.name;
```

**Batch by ids** (two queries, regardless of N)
```ts
const teams = await repo.teamsFor(orgId)
const counts = await repo.openTaskCounts(teams.map((t) => t.id))   // WHERE team_id = ANY($1) GROUP BY team_id
```

**ORM eager loading** (`include`, `JOIN FETCH`, `select_related`) — check the generated SQL; some
ORMs fetch eagerly with one extra query per relation, which is fine, others still loop.

**GraphQL dataloaders** — resolvers naturally produce N+1; a dataloader collects the ids requested
in one tick and issues one batched query.

## How to catch it

- Log SQL in development and count queries per request; fail a test if a route exceeds a budget.
- In production, traces show dozens of identical short spans under one request.
- `pg_stat_statements` shows a cheap query with an enormous call count.

## Pitfalls

- "Fixing" N+1 with caching — it hides the problem until the cache is cold.
- Loading everything eagerly everywhere — over-fetching is the opposite problem.

## Related

[Indexing](indexing.md) · [Aggregation](../sql/aggregation.md) · [Joins](../sql/joins.md)
