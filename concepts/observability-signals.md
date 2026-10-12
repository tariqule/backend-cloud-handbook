# Observability signals: logs, metrics, traces, RED and USE

> Logs tell you **what happened**, metrics tell you **how much and how fast**, traces tell you
> **where the time went** — correlated by a trace id.

[← Handbook](../README.md) · [All concepts](README.md)

---

## Logs

- **Structured JSON**, one event per line, to stdout.
- Every line carries `trace_id`/`request_id`, plus route, status, latency, tenant/user ids where relevant.
- Consistent levels: `error` (needs attention), `warn` (unexpected but handled), `info` (business events), `debug` (off in production).
- **Redact secrets by configuration** (tokens, `authorization`, cookies, passwords) — never rely on people remembering.

```json
{"level":"info","msg":"task assigned","trace_id":"4bf9…","route":"POST /v1/tasks/:id/assign","task_id":"t_9","latency_ms":42}
```

## Metrics

**RED** — for every service and endpoint:
- **R**ate — requests per second
- **E**rrors — failed requests per second (or as a ratio)
- **D**uration — latency **histograms** (p50, p95, p99)

**USE** — for every resource (CPU, memory, connections, queues):
- **U**tilization — how busy
- **S**aturation — how much work is waiting
- **E**rrors — resource errors

Never rely on averages for latency: a 50 ms average can hide a 3 s p99 that thousands of users feel.

### Cardinality

Each unique combination of metric labels creates a separate time series. Labels like `user_id`,
`task_id` or full URLs explode storage and cost. Keep metric labels **low-cardinality** (route
template, status class, region); put high-cardinality detail in logs and traces.

## Traces

A trace is a tree of **spans** — one per operation (HTTP handler, SQL query, outbound call, queue
publish) — sharing a trace id. Context crosses process boundaries in the W3C `traceparent` header
(and in message attributes for queues).

Instrument with **OpenTelemetry**: it auto-instruments common libraries (HTTP, Express, `pg`) and
exports to most backends, so you can change vendors — or clouds — without re-instrumenting.

## On GCP and AWS

| | Google Cloud | AWS |
|---|---|---|
| Logs | Cloud Logging | CloudWatch Logs |
| Metrics | Cloud Monitoring / Managed Service for Prometheus | CloudWatch / Amazon Managed Service for Prometheus |
| Traces | Cloud Trace | AWS X-Ray |
| OpenTelemetry | Supported natively | AWS Distro for OpenTelemetry |

## Related

[SLOs, error budgets and burn rate](slos-error-budgets-burn-rate.md) · [Event loop and libuv](event-loop-and-libuv.md)

## Further reading

- OpenTelemetry documentation: https://opentelemetry.io/docs/
- Brendan Gregg, *The USE Method*: https://www.brendangregg.com/usemethod.html
