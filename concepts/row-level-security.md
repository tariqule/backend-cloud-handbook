# Row-level security (PostgreSQL)

> Let the **database itself** filter which rows a session may see or change, so a forgotten
> `WHERE tenant_id = …` can't leak another customer's data.

[← Handbook](../README.md) · [All concepts](README.md)

---

## Why

In a multi-tenant app, every query on tenant data must filter by tenant. Hundreds of queries, one
forgotten filter, and customer A sees customer B's tasks. Application checks are necessary but
fragile. Row-level security (RLS) adds a **second, automatic layer**.

## Setting it up

```sql
ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;
ALTER TABLE tasks FORCE ROW LEVEL SECURITY;   -- apply even to the table owner

CREATE POLICY tenant_isolation ON tasks
  USING      (org_id = current_setting('app.org_id')::uuid)    -- which rows can be read/updated/deleted
  WITH CHECK (org_id = current_setting('app.org_id')::uuid);   -- which rows can be inserted/updated to
```

In the application, set the tenant **per transaction**:

```ts
await db.transaction(async (tx) => {
  await tx.$executeRaw`SELECT set_config('app.org_id', ${caller.orgId}, true)`   // true = local to this transaction
  return tx.task.findMany({ where: { status: 'open' } })   // RLS adds the org filter automatically
})
```

Using a **transaction-local** setting (`set_config(..., true)` / `SET LOCAL`) matters with
[connection pooling](connection-pooling.md): a session-level setting could leak to the next client
that gets the same pooled connection.

## Important details

- RLS doesn't apply to superusers or roles with `BYPASSRLS`. Run the app as a normal role.
- Without `FORCE ROW LEVEL SECURITY`, the **table owner** bypasses policies.
- If the setting is missing, `current_setting('app.org_id')` raises an error — good: fail closed.
  (`current_setting('app.org_id', true)` returns `NULL` instead, which also matches nothing.)
- Policies add a predicate to every query; index `org_id` (usually as the leading column).
- Background jobs and admin tools need a deliberate way to operate across tenants (a separate role
  with explicit, audited access).

## What it doesn't replace

RLS enforces **tenant boundaries**. Finer rules ("only the assignee can close this task") still
belong in your [authorization layer](authorization-models.md).

## Related

[Authorization models](authorization-models.md) · [Connection pooling](connection-pooling.md)

## Further reading

- PostgreSQL docs, *Row Security Policies*: https://www.postgresql.org/docs/current/ddl-rowsecurity.html
