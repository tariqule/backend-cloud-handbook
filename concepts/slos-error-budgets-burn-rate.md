# SLOs, error budgets and burn-rate alerts

> Define reliability as a measurable target, treat the allowed failure as a **budget**, and alert
> when you're **spending it too fast**.

[← Handbook](../README.md) · [All concepts](README.md)

---

## The vocabulary

| Term | Meaning | Example |
|---|---|---|
| **SLI** — indicator | What you measure, as good events ÷ valid events | Requests answered successfully in < 300 ms ÷ all requests |
| **SLO** — objective | Your target for the SLI over a window | 99.9% over 30 days |
| **SLA** — agreement | A contract with consequences; looser than the SLO | 99.5%, with service credits |
| **Error budget** | 100% − SLO | 0.1% of requests may fail |

## How big is a budget?

| SLO | Allowed "bad" time per 30 days |
|---|---|
| 99% | ~7.2 hours |
| 99.5% | ~3.6 hours |
| 99.9% | ~43 minutes |
| 99.95% | ~22 minutes |
| 99.99% | ~4.3 minutes |

Each extra nine costs roughly ten times more engineering. Choose the lowest SLO users are happy with.

## Using the budget

- Budget left → ship features, run experiments.
- Budget nearly spent → slow down, prioritize reliability work.
- It turns "is it reliable enough?" from an argument into a number.

## Burn rate

**Burn rate** = how fast you're spending the budget relative to plan.
- Burn rate **1** spends exactly the whole budget over the window.
- Burn rate **14.4** sustained for **1 hour** spends **2%** of a 30-day budget
  (14.4 × 1 h ÷ 720 h = 0.02).

## Multi-window, multi-burn-rate alerts

Static thresholds ("error rate > 1%") are noisy for brief spikes and slow for gradual problems.
The approach from Google's SRE Workbook pages on fast burns and opens tickets for slow ones, and
requires a short window to confirm the problem is still happening:

| Severity | Long window | Short window | Burn rate | Budget consumed |
|---|---|---|---|---|
| Page | 1 hour | 5 minutes | 14.4 | 2% |
| Page | 6 hours | 30 minutes | 6 | 5% |
| Ticket | 3 days | 6 hours | 1 | 10% |

An alert fires only when **both** windows exceed the burn rate — the long window proves it's
significant, the short one proves it's still going on.

## Pitfalls

- SLOs nobody looks at, or that don't reflect what users feel.
- Measuring only server-side success while users see failures upstream (edge, DNS, client).
- Aiming for 100% — impossible, and it freezes delivery.

## Related

[Circuit breakers, bulkheads and load shedding](circuit-breakers-bulkheads-load-shedding.md) ·
[Observability signals](observability-signals.md)

## Further reading

- Google SRE Workbook, *Alerting on SLOs*: https://sre.google/workbook/alerting-on-slos/
- Google SRE Book, *Service Level Objectives*: https://sre.google/sre-book/service-level-objectives/
