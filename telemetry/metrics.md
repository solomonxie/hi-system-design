# Metrics

## The three metric types

- **Counter**: monotonically increasing value (`requests_total`, `errors_total`). Only useful as
  a rate (`rate(requests_total[5m])`) — the raw cumulative number resets on restart and isn't
  meaningful on its own.
- **Gauge**: a value that goes up or down (`queue_depth`, `active_connections`, `memory_used`).
  Point-in-time state, not a rate.
- **Histogram** (or summary): distribution of observed values (`request_duration_seconds`) —
  bucketed counts you can derive percentiles from. This is how you get p50/p95/p99 latency rather
  than just an average, which hides tail behavior entirely.

## Why percentiles, not averages

An average latency of 100ms can hide a p99 of 5 seconds if most requests are fast and a small
tail is very slow — and that tail is usually exactly the requests that matter (they're often
correlated: same large customer, same unindexed query, same cold cache). Alert and SLO on p95/p99,
not mean, for anything user-facing.

## The four golden signals (Google SRE)

Latency, traffic, errors, saturation. A minimal but genuinely load-bearing dashboard for any
service covers all four:

- **Latency**: p50/p95/p99, split by success vs. error (a fast error and a slow success average
  to something meaningless together).
- **Traffic**: requests/sec, or whatever the natural demand unit is for this system.
- **Errors**: rate of failed requests, ideally broken down by failure reason/type.
- **Saturation**: how "full" the system is relative to its limit (CPU, memory, connection pool,
  queue depth) — the leading indicator of the other three getting worse soon.

## Cardinality: the recurring cost trap

A metric's cost scales with the number of unique label-value combinations (its cardinality), not
just the number of metrics. A label like `user_id` or raw `url` on a per-request metric can
create millions of unique time series and take down a Prometheus instance or blow up a hosted
metrics bill. Keep labels to bounded, low-cardinality dimensions (`status_code`, `region`,
`endpoint_template` — not the raw URL with path params filled in). Put anything genuinely
per-entity in traces or logs instead.

## RED and USE, as quick mnemonics

- **RED** (per request-driven service): Rate, Errors, Duration.
- **USE** (per resource — CPU, disk, queue): Utilization, Saturation, Errors.

## Staff-engineer notes

- Define SLOs (Service Level Objectives) as a percentile over a time window ("99% of requests
  under 300ms over a rolling 28 days"), not a hard pass/fail per request — this is what an error
  budget is built on, and what actually drives a sane alerting threshold instead of an arbitrary
  number someone picked once.
- A metric with no owner and no alert tied to it is dashboard decoration — it will not be
  noticed when it degrades. Every metric worth collecting is worth deciding, up front, whether
  it needs an alert.
- When proposing a new service in an RFC, name its SLIs (the metrics that define "healthy") in
  the RFC itself, before it ships — retrofitting this after an incident is a worse conversation
  to have.
