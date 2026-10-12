# The transactional outbox pattern

> Publish events reliably by **writing them to a database table in the same transaction** as the
> business change, then relaying them to the message broker afterwards.

[← Handbook](../README.md) · [All concepts](README.md)

---

## The dual-write problem

A use case must update the database **and** publish an event ("task assigned"). Two systems, no
shared transaction:

| Order | Failure | Result |
|---|---|---|
| Commit, then publish | Crash after commit | Data changed, **event lost** |
| Publish, then commit | Commit fails | **Event sent for a change that never happened** |

There's no ordering that's safe.

## The pattern

```
┌─────────────── one transaction ───────────────┐
│ UPDATE tasks SET assignee_id = …               │
│ INSERT INTO outbox (event_id, type, payload)   │
└────────────────────────────────────────────────┘
        relay (polling or CDC) ──► broker ──► consumers (idempotent)
```

```sql
CREATE TABLE outbox (
  id           bigserial   PRIMARY KEY,
  event_id     uuid        NOT NULL UNIQUE,
  type         text        NOT NULL,
  payload      jsonb       NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  published_at timestamptz
);
CREATE INDEX outbox_unpublished ON outbox (id) WHERE published_at IS NULL;
```

**Relay with polling** (several relays can run safely):
```sql
WITH batch AS (
  SELECT id FROM outbox WHERE published_at IS NULL
  ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 100
)
SELECT o.* FROM outbox o JOIN batch b USING (id);
-- publish each to the broker, then:
UPDATE outbox SET published_at = now() WHERE id = ANY($1);
```

**Relay with CDC** — a change-data-capture tool (Debezium, or a managed service such as Datastream
on GCP or DMS on AWS) tails the database log and publishes outbox inserts.

## Guarantees

- An event exists **if and only if** the business change committed.
- Delivery is **at least once** (the relay may crash after publishing but before marking it sent),
  so consumers must be [idempotent](idempotency.md), deduplicating by `event_id`.
- Ordering: by outbox `id` per relay; across relays or partitions, design consumers to tolerate reordering.

## Operating it

- Alert on the age of the oldest unpublished row (relay lag).
- Delete or archive published rows on a schedule to keep the table small.
- Keep payloads versioned and additive — events are contracts.

## Related

[Idempotency](idempotency.md) · [Sagas](sagas.md) · [Locking and SKIP LOCKED](../sql/locking-and-skip-locked.md)

## Further reading

- microservices.io, *Transactional outbox*: https://microservices.io/patterns/data/transactional-outbox.html
