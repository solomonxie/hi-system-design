# ELK / Elastic Stack (Elasticsearch, Logstash, Kibana, Beats)

The standard open-source logging/search stack. Elasticsearch stores and indexes the data,
Logstash/Beats get it there, Kibana is the query/dashboard UI on top.

## The pieces

- **Beats**: lightweight, single-purpose shippers that run alongside the source (Filebeat for log
  files, Metricbeat for host/service metrics, Packetbeat for network data) and forward to
  Logstash or directly to Elasticsearch. Low resource footprint by design — meant to sit on every
  host without competing for its resources.
- **Logstash**: the heavier-weight ingest pipeline — parses, transforms, and enriches events
  (grok patterns to parse unstructured log lines into structured fields, adding geo lookups,
  dropping noisy fields) before indexing. Increasingly, simple parsing is pushed to Beats or done
  at the application layer (structured logging from the start, see `logging.md`) so Logstash's
  transform stage has less work to do.
- **Elasticsearch**: a distributed document store built on an inverted index — this is what makes
  full-text and structured search over huge log volumes fast, at the cost of being a genuinely
  complex distributed system to operate (shard/replica management, cluster sizing, hot-warm-cold
  tiering for cost control on older data).
- **Kibana**: search UI, dashboards, and (via the Elastic Observability/Security apps) APM and
  SIEM-style features built on top of the same Elasticsearch data.

## Where it fits vs. the metrics stack

ELK is optimized for high-cardinality, exploratory search over discrete events (find every log
line matching this trace ID, this error message, this user) — the thing Prometheus explicitly
isn't built for. Many real stacks run both: Prometheus/Grafana for metrics and alerting,
Elasticsearch/Kibana (or the Loki equivalent, which trades full-text indexing for much cheaper
storage by only indexing labels) for log search.

## Staff-engineer notes

- Elasticsearch cluster health (shard allocation, disk watermarks) is itself something that needs
  monitoring — a red/yellow cluster state silently degrades search and can start rejecting writes,
  which then means the logging pipeline itself is dropping data during exactly the kind of
  high-traffic/incident period you need it most.
- Structure logs at the source (see `logging.md`) rather than leaning on Logstash grok patterns
  to parse free text after the fact — grok patterns are brittle (a log format change silently
  breaks parsing) and push the cost of unstructured logging downstream instead of removing it.
- Index/retention policy (ILM — index lifecycle management: hot → warm → cold → delete) needs to
  be a deliberate decision tied to actual query/compliance needs, not "whatever the default is" —
  unmanaged log retention is one of the most common runaway infra cost stories.
