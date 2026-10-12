# Streams and backpressure

> Process large data **piece by piece**, and let a slow consumer **slow down the producer** instead
> of buffering everything in memory.

[← Handbook](../README.md) · [All concepts](README.md)

---

## The problem

```ts
const rows = await db.query('SELECT * FROM tasks WHERE org_id = $1', [orgId])   // 2 million rows in memory
res.json(rows)                                                                   // one giant string in memory
```

Memory spikes, GC pauses, the event loop blocks while stringifying, and under concurrent exports
the container is OOM-killed.

## Streams

A stream emits data in chunks. Piping streams together lets data flow through as it's produced:

```ts
import { pipeline } from 'node:stream/promises'
import { createGzip } from 'node:zlib'
import QueryStream from 'pg-query-stream'

const query = client.query(new QueryStream('SELECT id, title, status FROM tasks WHERE org_id = $1', [orgId]))
res.setHeader('Content-Type', 'text/csv')
res.setHeader('Content-Encoding', 'gzip')
await pipeline(query, toCsvTransform(), createGzip(), res)
```

Memory stays flat no matter how many rows there are.

## Backpressure

Each writable stream has a buffer limit (`highWaterMark`). When it's full, `write()` returns
`false` and the producer should pause until `'drain'`. `pipeline()` (and `pipe()`) handle this for
you: if the client downloads slowly, reading from the database slows down too.

If you write manually, respect it:
```ts
if (!res.write(chunk)) await once(res, 'drain')
```

Ignoring `write()`'s return value means data piles up in memory — the same problem streams were
supposed to solve.

## Why `pipeline()` and not `.pipe()`

`pipeline()` forwards errors and **destroys every stream** when one fails (for example the client
disconnects), so database cursors and file handles don't leak. `.pipe()` doesn't propagate errors.

## Beyond HTTP responses

The same idea applies everywhere: reading uploads to object storage, processing files line by line,
consuming queues with a bounded number of in-flight messages, and limiting concurrency
(`p-limit`) instead of `Promise.all` over thousands of items.

## Related

[Event loop and libuv](event-loop-and-libuv.md) · [Keyset pagination](keyset-pagination.md)

## Further reading

- Node.js, *Backpressuring in Streams*: https://nodejs.org/en/learn/modules/backpressuring-in-streams
- Node.js API docs, `stream.pipeline`: https://nodejs.org/api/stream.html
