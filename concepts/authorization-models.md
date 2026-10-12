# Authorization models: RBAC, ABAC and ReBAC

> Authentication answers *who are you*; authorization answers *may you do this, to this object,
> right now*.

[← Handbook](../README.md) · [All concepts](README.md)

---

## The three models

| Model | Decision based on | Example rule | Fits |
|---|---|---|---|
| **RBAC** — role-based | The caller's roles | "Admins can delete teams" | Coarse, org-wide permissions |
| **ABAC** — attribute-based | Attributes of caller, resource and context | "Members can edit tasks in their own team during business hours" | Rules that depend on data |
| **ReBAC** — relationship-based | A graph of relationships | "You can view a document if you're in a group it's shared with, or own its folder" | Sharing, hierarchies, collaboration |

Real products usually combine them: **RBAC inside a tenant** plus **per-object checks**.

## ReBAC in one picture

```
user:alice ──member──► team:design ──viewer──► project:website ──parent──► task:t_9
```
"Can Alice view task t_9?" → follow the edges. Systems based on Google's **Zanzibar** paper
(OpenFGA, SpiceDB, Permify) store these relationships and answer checks fast and consistently.

## Make authorization structural

- **One decision function per action** in the service layer — `canEditTask(actor, task)` — called
  by every route that edits tasks. No ad-hoc `if (user.role === 'admin')` scattered around.
- **Check the object, not just the role.** "Logged in" and "is a member somewhere" aren't enough;
  is *this* task in a team *this* caller belongs to? Missing this is the classic **IDOR** bug.
- **Deny by default.** No rule allows it → forbidden.
- **Hide what callers shouldn't know exists:** return `404`, not `403`, for other tenants' resources.
- **Never trust ids, roles or tenant ids from the request body** — derive them from the
  authenticated identity.
- **Enforce tenant isolation in the data layer** too ([row-level security](row-level-security.md)).

## Testing

For every route, test: no identity, wrong role, another tenant's object, another user's object,
and the allowed case. A table-driven test makes the matrix cheap to maintain.

## Pitfalls

- Authorization only in the UI or only in one of two routes that reach the same data.
- Roles cached in long-lived tokens, so removed users keep access.
- Admin "impersonation" features without audit logging.

## Related

[JWTs and token verification](jwt-and-token-verification.md) · [Row-level security](row-level-security.md) ·
[Least privilege](least-privilege-iam.md)

## Further reading

- OWASP, *Authorization Cheat Sheet*: https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html
- *Zanzibar: Google's Consistent, Global Authorization System*: https://research.google/pubs/zanzibar-googles-consistent-global-authorization-system/
