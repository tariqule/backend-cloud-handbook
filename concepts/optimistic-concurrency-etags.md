# Optimistic concurrency and ETags

> Let concurrent edits proceed without locks, and **reject the later write if the data changed
> since it was read**.

[← Handbook](../README.md) · [All concepts](README.md)

---

## The problem: lost updates

Two people open the same task. Alice changes the assignee; Bob changes the due date from the
version he loaded earlier. Bob saves last and his request — carrying the old assignee — silently
overwrites Alice's change. Nobody sees an error.

## The idea

Give every row a **version**. A write states which version it was based on. If the row's version
has moved on, the write fails and the client must reload.

```sql
ALTER TABLE tasks ADD COLUMN version int NOT NULL DEFAULT 1;

UPDATE tasks
SET title = $3, version = version + 1, updated_at = now()
WHERE id = $1 AND version = $2
RETURNING *;
-- 0 rows updated → someone else changed it first
```

It's "optimistic" because it assumes conflicts are rare and only checks at write time, instead of
locking rows while a user is editing.

## Over HTTP: ETags and conditional requests

| Step | Request / response |
|---|---|
| Read | `GET /tasks/t_9` → `200`, `ETag: "v7"` |
| Update | `PATCH /tasks/t_9` with `If-Match: "v7"` |
| Success | `200`, `ETag: "v8"` |
| Conflict | `412 Precondition Failed` — reload and retry |

The same header family also saves bandwidth on reads:

| Read | `GET /tasks/t_9` with `If-None-Match: "v8"` → `304 Not Modified` (no body) |
|---|---|

```ts
router.patch('/tasks/:id', async (req, res) => {
  const expected = parseEtag(req.header('if-match'))           // "v7" → 7
  if (expected === undefined) throw errors.preconditionRequired()   // 428
  const task = await tasks.update(req.user, req.params.id, req.body, expected)
  res.set('ETag', `"v${task.version}"`).json(task)
})
// In the repository: zero rows updated → throw a PreconditionFailed error → 412
```

## Optimistic vs pessimistic

| | Optimistic (versions) | Pessimistic (`SELECT … FOR UPDATE`) |
|---|---|---|
| Holds locks | No | Yes, for the transaction |
| Best when | Conflicts are rare; edits span user think-time | Conflicts are frequent; one short transaction |
| On conflict | Client reloads and retries | Second transaction waits |

Never hold a database lock while a human is editing a form — use versions.

## Pitfalls

- Making `If-Match` optional, so old clients bypass the check. Consider `428 Precondition Required`.
- Weak ETags (`W/"…"`) are fine for caching but not for `If-Match` on writes.
- Bumping the version only in some code paths — every write must increment it.

## Related

[Isolation levels and write skew](isolation-levels-and-write-skew.md) · [Idempotency](idempotency.md) ·
[Locking and SKIP LOCKED](../sql/locking-and-skip-locked.md)

## Further reading

- RFC 9110 §13, *Conditional Requests*: https://www.rfc-editor.org/rfc/rfc9110
