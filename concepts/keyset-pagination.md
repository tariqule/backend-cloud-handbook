# Keyset (cursor) pagination

> Page through results by remembering **where the last page ended**, not how many rows to skip.

[← Handbook](../README.md) · [All concepts](README.md)

---

## Why not `OFFSET`?

```sql
SELECT * FROM tasks ORDER BY created_at DESC LIMIT 50 OFFSET 10000;
```

- **Slow:** the database still reads and throws away 10,000 rows. Deep pages get slower and slower.
- **Unstable:** if a new task arrives while someone is paging, every row shifts by one — they see a
  duplicate at the top of the next page (or miss a row when items are deleted).

## Keyset pagination

Remember the sort key of the last row and continue from there:

```sql
-- First page
SELECT id, title, created_at FROM tasks
WHERE team_id = $1
ORDER BY created_at DESC, id DESC
LIMIT 51;            -- fetch one extra to know if there's a next page

-- Next page: continue after the last row (created_at = $2, id = $3)
SELECT id, title, created_at FROM tasks
WHERE team_id = $1
  AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC
LIMIT 51;
```

With an index on `(team_id, created_at DESC, id DESC)`, every page is a short index range scan —
page 1 and page 10,000 cost the same.

## The tie-breaker is not optional

Timestamps collide. Ordering by `created_at` alone means two rows with the same timestamp at a page
boundary can be skipped or repeated. Always add a **unique** column (usually `id`) as the last sort
key, and compare with a **row-value comparison** `(created_at, id) < ($2, $3)`.

## The cursor

Encode the last row's keys opaquely so clients can't depend on its format:

```ts
const cursor = Buffer.from(JSON.stringify({ c: last.createdAt, i: last.id })).toString('base64url')
// Response
{ "data": [ /* 50 items */ ], "page": { "nextCursor": "eyJjIjoiMjAyNi0…" } }
```

`nextCursor` is `null` on the last page (when the extra 51st row wasn't returned).

## Trade-offs

| | Offset | Keyset |
|---|---|---|
| Jump to page N | ✅ | ❌ (next/previous only) |
| Deep pages | Slow | Fast |
| Stable under inserts | ❌ | ✅ |
| Total count | Often expected | Optional, and expensive on big sets |

Use offset for small admin tables where "page 7" matters; keyset for feeds, APIs and anything large.

## Related

[Indexing](indexing.md) · [Row-value comparisons and gotchas](../sql/nulls-and-gotchas.md)

## Further reading

- *Use The Index, Luke* — "Paging Through Results": https://use-the-index-luke.com/no-offset
