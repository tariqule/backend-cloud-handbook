# Idempotency

> An operation is **idempotent** if doing it once or many times has the same effect.

[← Handbook](../README.md) · [All concepts](README.md)

---

## Why it matters

Networks fail in ways that hide the outcome. A client sends "charge $50", the connection times
out, and the client can't tell whether the server charged the card. The only safe move is to
retry — so the server must make retries harmless. The same applies to message consumers: most
queues deliver **at least once**, so a consumer will eventually see the same message twice.

## Idempotent by nature vs made idempotent

| Operation | Naturally idempotent? |
|---|---|
| `GET /tasks/1` | ✅ reads change nothing |
| `PUT /tasks/1 { "title": "A" }` | ✅ setting a value twice leaves the same value |
| `DELETE /tasks/1` | ✅ the task is gone either way (second call may return 404) |
| `UPDATE … SET status = 'done'` | ✅ |
| `UPDATE … SET count = count + 1` | ❌ each call adds one |
| `POST /payments` | ❌ each call creates a payment |
| "send an email" | ❌ each call sends one |

Non-idempotent operations need an **idempotency key**.

## How idempotency keys work

1. The client generates a unique key per *logical* operation (a UUID) and sends it with every retry:
   `Idempotency-Key: 5f2b…`.
2. The server stores `(key, caller, request fingerprint) → response` **in the same transaction** as
   the side effect.
3. A repeat with the same key and same body returns the stored response without redoing the work.
4. A repeat with the same key but a different body is a client bug → reject (`422` or `409`).
5. Keys expire after a window (often 24 hours).

```sql
CREATE TABLE idempotency_keys (
  caller_id     uuid        NOT NULL,
  key           text        NOT NULL,
  request_hash  text        NOT NULL,
  response_code int         NOT NULL,
  response_body jsonb       NOT NULL,
  created_at    timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (caller_id, key)
);
```

```ts
await db.transaction(async (tx) => {
  const existing = await tx.idempotency.find(caller.id, key)
  if (existing) {
    if (existing.requestHash !== hash(body)) throw errors.conflict('Key reused with a different request')
    return existing.response
  }
  const payment = await tx.payments.create(body)          // the side effect
  const response = { status: 201, body: payment }
  await tx.idempotency.save(caller.id, key, hash(body), response)   // same transaction
  return response
})
```

The primary key also handles two *concurrent* requests with the same key: one insert wins, the
other gets a unique violation and can re-read the stored response.

## Idempotent consumers

For queues, use the message or event id as the key:

```sql
CREATE TABLE processed_events (event_id uuid PRIMARY KEY, processed_at timestamptz DEFAULT now());
```

Insert into `processed_events` in the same transaction as the consumer's database changes. A
redelivered message hits the primary key and is skipped.

## Pitfalls

- **Storing the key after the side effect commits.** A crash between the two loses the record and
  the retry repeats the work. Store both in one transaction.
- **Keys generated per attempt** instead of per operation — every retry looks new.
- **Side effects outside the database** (emails, third-party APIs). Pass your idempotency key to
  the third party if it supports one, or record "about to send" / "sent" states.
- **Retries at every layer.** Idempotency makes retries safe, not cheap — see
  [retries and backoff](retries-backoff-jitter.md).

## Related

[Retries, backoff and jitter](retries-backoff-jitter.md) · [Outbox pattern](outbox-pattern.md) ·
[Optimistic concurrency and ETags](optimistic-concurrency-etags.md)

## Further reading

- IETF draft, *The Idempotency-Key HTTP Header Field*: https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/
- RFC 9110 §9.2.2, *Idempotent Methods*: https://www.rfc-editor.org/rfc/rfc9110
