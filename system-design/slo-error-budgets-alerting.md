# SLOs, error budgets and alert priorities

How to state a reliability target, how much failure it permits, and which problems deserve to wake someone. Docs-based. Overview of all three signals: [[observability]]; siblings: [[logging]], [[metrics-prometheus]], [[distributed-tracing]].

## SLI, SLO, SLA

| | What it is | Example | Who it is for |
|---|---|---|---|
| **SLI** — indicator | A measurement of service quality | Share of requests answered without a server error in under 300 ms | Engineers |
| **SLO** — objective | The target for that measurement over a period | 99.9% over a rolling 30 days | The team and its stakeholders |
| **SLA** — agreement | A contract with a customer, with consequences if missed | 99.5% per month, or service credits | Customers and lawyers |

- The chain runs one way: an SLI is measured, an SLO sets a target on it, an SLA makes a weaker version of that target a commercial promise.
- **The SLO is always stricter than the SLA.** The gap is the margin for noticing and fixing a problem before it costs money.
- An SLA without an internal SLO means the first signal of trouble is a contractual breach.

### Choosing SLIs

- Express it as a ratio: **good events ÷ valid events**. "Requests that succeeded quickly ÷ all requests" works for availability and latency alike, and always reads as a percentage.
- Measure **where the user is**: at the load balancer, the CDN, or with synthetic journeys from outside. A measurement taken inside the service misses DNS, certificate and network failures — the service reports 100% while nobody can reach it.
- Keep it to a few per service — availability and latency, plus freshness or correctness for data pipelines. A critical journey (checkout, login) deserves its own.
- Decide what counts as valid: client errors caused by bad input usually do not count against the service; health checks and load tests are excluded.

### Choosing the target

- **100% is never the target.** It is unreachable, forbids every change, and users could not tell the difference — their own network is less reliable than four nines.
- Each additional nine costs roughly an order of magnitude more in redundancy, automation and engineering time.
- A service cannot be more reliable than the critical dependencies it calls in series. An SLO above theirs is a wish.
- Start from what users currently get and tolerate, not from an aspiration.

## Error budget

```
Error budget = 100% − SLO
```

An SLO of 99.9% leaves a budget of 0.1%: the amount of failure that is **allowed**.

| SLO | Budget | Downtime per 30 days | Per year |
|---|---|---|---|
| 99% | 1% | 7 h 12 min | 3.65 days |
| 99.5% | 0.5% | 3 h 36 min | 1.83 days |
| 99.9% | 0.1% | 43 min | 8.8 hours |
| 99.95% | 0.05% | 22 min | 4.4 hours |
| 99.99% | 0.01% | 4 min 19 s | 53 minutes |
| 99.999% | 0.001% | 26 s | 5 minutes |

In requests rather than time: a service handling 10 million requests a month at 99.9% may fail 10,000 of them.

Why it matters:

- **It turns reliability into a quantity that can be spent.** Deployments, experiments, migrations and planned maintenance all consume budget; so do incidents. While budget remains, shipping fast is by definition acceptable.
- **It settles the argument between speed and stability with a number** rather than opinion. Budget left: release. Budget gone: stop feature releases and work on reliability until it recovers.
- That rule must be written down and agreed in advance — the **error budget policy**. An SLO with no consequence for exhausting the budget changes nothing.
- A budget that is never touched is also information: the target is looser than the service, or the team is being more cautious than it needs to be.

### Burn rate

**Burn rate** is how fast the budget is being consumed relative to the pace that would use exactly all of it by the end of the period. Burn rate 1 lasts the full 30 days; burn rate 10 exhausts it in three.

It is the basis for alerts that scale with impact:

| Burn rate | Over | Budget consumed | Response |
|---|---|---|---|
| 14.4 | 1 hour | 2% | Page now |
| 6 | 6 hours | 5% | Page |
| 3 | 1 day | 10% | Ticket |
| 1 | 3 days | 10% | Ticket |

Each is checked together with a short window (for example the last five minutes) so the alert clears soon after the problem stops. A brief total outage and a long slight degradation are both caught, and a small blip that costs almost no budget pages nobody.

## Alerts

### Priorities

Names and exact times differ between organizations; the shape is constant.

| Priority | Situation | Examples | Response |
|---|---|---|---|
| **P1** — critical | Users are affected now, widely, or data or security is at risk | Site down; checkout failing; data loss; fast budget burn | Page immediately, any hour; acknowledge within minutes; incident declared |
| **P2** — high | Partial or degraded service, or protection lost with users not yet affected | One region failing over; elevated error rate on one feature; a replica down leaving no redundancy; slow budget burn | Notify on-call; act within hours, in working hours unless it worsens |
| **P3** — low | No user impact; will become a problem if ignored | Disk 70% full; certificate expiring in 30 days; a retried background job; a flaky non-critical check | Ticket; handle in normal work within days |

- **Only P1, and P2 that is getting worse, interrupt a person.** Everything else goes to a queue.
- The same condition changes priority with time: a certificate expiring in 30 days is P3, in 3 days P1.
- **Severity and priority are different axes** in many schemes — severity is how bad the impact is, priority is how soon it is worked on. A severe problem with a working workaround can be lower priority.
- A priority is a commitment about response. If P1 fires weekly for things that can wait, it has stopped meaning P1.

### What makes an alert good

- **Alert on symptoms, diagnose with causes.** "Users are getting errors" pages; "CPU is at 90%" is a dashboard. Cause-based pages fire when no user is affected and stay silent for failures nobody predicted.
- **Actionable**: someone must be able to do something now. If the only possible response is to watch, it is not a page.
- **It carries what is needed**: what is broken, how badly, since when, a link to the dashboard and to the **runbook**.
- **It has an owner** — one team, not a shared channel where everyone assumes someone else has it.
- **It tolerates blips**: a condition must hold for a few minutes before firing, and related alerts are grouped so one failure is one notification, not two hundred.
- **Absence of data is alertable.** A job that stops reporting, a metric that disappears: use a heartbeat that alerts when it is *not* received.

```yaml
groups:
  - name: orders-slo
    rules:
      - alert: OrdersErrorBudgetFastBurn
        expr: |
          sum(rate(http_requests_total{service="orders",status=~"5.."}[1h]))
            / sum(rate(http_requests_total{service="orders"}[1h])) > (14.4 * 0.001)
        for: 2m
        labels:
          severity: P1
        annotations:
          summary: "orders is burning its 30-day error budget 14x too fast"
          runbook: "https://runbooks.example.com/orders/error-budget"
```

The `severity` label is what the alert router uses to choose between paging, a chat message and a ticket.

### Traps

- **Alert fatigue** is a reliability risk of its own: a pager that fires for nothing is ignored when it fires for something. Review every page — was action needed? If not, fix or delete the alert.
- **Static thresholds** on resource metrics need retuning forever and still miss real incidents. Burn-rate alerts on SLOs replace most of them.
- **Alerting on every error** rather than on the rate against the budget.
- **No escalation**: an unacknowledged P1 must go to a second person automatically.
- **Monitoring that shares the fate of what it monitors** — the alerting system in the same cluster or region as the service fails with it. Have an external check on the monitoring itself.

## What this means for AEM

- **AEMaaCS comes with an SLA from Adobe** (99.9% as standard, 99.99% as a purchasable option — [cloud-manager-program-types](../cloud-service/cloud-manager-program-types.md)). That is Adobe's promise about the platform, not an SLO for the site: custom code, third-party integrations and CDN configuration sit in front of users too and are not covered by it.
- A customer-side SLO is therefore measured from outside — synthetic checks on key pages and CDN logs — and its error budget is mostly spent by the customer's own deployments and integrations.
