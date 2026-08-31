# Prometheus & Grafana

The standard open-source metrics stack: Prometheus collects and stores time-series metrics and
evaluates alert rules; Grafana visualizes them (and much more than just Prometheus — it can query
most metrics/logs/trace backends behind a common dashboard).

## Prometheus's model

- **Pull, not push**: Prometheus scrapes each target's `/metrics` HTTP endpoint on an interval,
  rather than services pushing metrics to it. This makes "is this service reachable at all"
  itself an implicit health signal (a failed scrape is visible), and keeps instrumentation
  simple (expose a text endpoint, no client-side batching/retry logic needed). Short-lived jobs
  that don't live long enough to be scraped push through the **Pushgateway** instead — the
  documented exception, not the default.
- **PromQL**: the query language for aggregating/filtering time series — e.g.
  `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` for p99 latency
  over a 5-minute window. Worth actually learning `rate()`, `histogram_quantile()`, and label
  matching (`{status_code=~"5.."}`) — most dashboards and alerts are built from a handful of
  these patterns repeated.
- **Local storage, limited retention by design**: Prometheus's own TSDB is built for
  recent-window operational queries (typically weeks), not long-term/historical storage — for
  that, remote-write to a long-term backend (Thanos, Cortex, Mimir, a hosted vendor).
- **Alertmanager**: separate component that receives firing alerts from Prometheus, handles
  deduplication, grouping, silencing, and routing to notification channels (PagerDuty, Slack,
  email) — alerting rules live in Prometheus, routing/on-call logic lives in Alertmanager.

## Grafana's role

A dashboard and alerting UI that queries Prometheus (and Loki for logs, Tempo/Jaeger for traces,
Elasticsearch, CloudWatch, and many more) through data source plugins — one pane of glass across
otherwise separate backends. Dashboards are usually version-controlled as JSON (or built via
Terraform/grafonnet) rather than clicked together by hand, so they survive a Grafana instance
being rebuilt and get reviewed like code.

## Staff-engineer notes

- Prometheus's pull model plus service discovery (Kubernetes, Consul, EC2 tags) means new
  instances get scraped automatically as they come up — but it also means a target that's simply
  slow to start or misconfigured for discovery silently produces no data rather than an obvious
  error; check "is this target even being scraped" before debugging "why is this metric wrong."
- Cardinality bites Prometheus specifically hard: each unique label combination is a separate
  time series stored in memory, and an unbounded label (see `metrics.md`) can OOM a Prometheus
  instance, not just cost more — this is an operational incident, not just a bill.
- Treat dashboards and alert rules as code (checked into the same repo/review process as the
  service they monitor) — a dashboard that only exists as manual Grafana UI state gets stale and
  eventually diverges from what the service actually does.
