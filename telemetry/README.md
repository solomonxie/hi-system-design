# Telemetry & Observability

The three pillars — logs, metrics, traces — plus the platforms that collect, store, and surface
them. Observability isn't "we have a dashboard"; it's the ability to answer a question you
didn't anticipate asking, about a system state you didn't anticipate happening, without shipping
new code first. That last clause is the actual bar: if answering a new question requires a
deploy, you have monitoring, not observability.

## The three pillars, in one line each

- **Logs**: discrete, timestamped events with context — the detailed "what happened" record.
  Best for deep-diving one specific request/entity once you already know where to look.
- **Metrics**: numeric measurements aggregated over time (counters, gauges, histograms) — cheap
  to store at high cardinality of *time*, expensive at high cardinality of *labels*. Best for
  "is the system healthy right now" and alerting.
- **Traces**: the path of one request across services/functions, as a tree of timed spans. Best
  for "where did the time go" and "which downstream call actually failed" in a distributed call
  chain.

None of the three substitutes for the others — metrics tell you *that* p99 latency spiked,
traces tell you *which service* in the chain caused it, logs tell you *why* (the actual
exception, the actual input). See `logging.md`, `metrics.md`, `tracing.md`.

## Platforms in this folder

- `tracing.md` — OpenTelemetry: the vendor-neutral instrumentation standard (API + SDK + OTLP
  wire protocol) that emits all three signal types to whatever backend you choose.
- `prometheus-grafana.md` — the standard OSS metrics stack: Prometheus (pull-based scraping +
  storage + alerting) and Grafana (dashboards, works with far more than just Prometheus).
- `kibana-elk.md` — the standard OSS logging stack: Elasticsearch (storage/search),
  Logstash/Beats (ingestion), Kibana (dashboards/search UI).

## Staff-engineer notes

- Instrument for the question you'll actually ask during an incident ("which customer, which
  region, which code path"), not just for a green dashboard. A service with 100% uptime on its
  dashboard and no way to tell which tenant is degraded is not actually observable.
- Cardinality is the recurring cost trap: a metric label or log field with unbounded values
  (user ID, request ID, raw URL with query params) blows up storage/query cost in metrics systems
  and can silently make a Prometheus instance fall over. Put unbounded identifiers in traces/logs
  (built for high cardinality), not metric labels.
- Correlation IDs (a request/trace ID threaded through every log line, every span, every queue
  message) are the single highest-leverage observability investment — without one, "logs" and
  "traces" for the same request are unrelated data you have to correlate by eyeballing
  timestamps.
- Alert on symptoms (latency, error rate, saturation — the user-facing effect), not on causes
  (CPU%, a specific queue depth) except where a cause metric is a reliable, well-understood
  leading indicator you've validated against real incidents. Cause-based alerts rot fast as the
  system changes underneath them.
