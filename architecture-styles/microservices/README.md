# Microservices

Independently deployable services, each owning its own data store, communicating over the
network (REST/gRPC for request/response, events/queues for async). Solves an organizational
scaling problem — many teams needing independent deploy cadence — at the cost of a full
distributed-systems tax.

## What you actually take on

- **Network is now in the critical path** for calls that used to be a function call: latency,
  timeouts, retries, partial failure. Every inter-service call needs a timeout and a fallback
  behavior for "the other service is slow or down" (see circuit breakers below).
- **Data consistency across services** — no more single-database ACID transaction spanning the
  whole request. Cross-service writes need sagas, outbox patterns, or eventual consistency (see
  `distributed-systems/distributed-transactions.md`).
- **Observability becomes mandatory, not optional** — a single user request now touches N
  services; without distributed tracing (see `telemetry/`) you cannot answer "why was this
  request slow" at all.
- **Deployment and infra multiply**: N services means N CI/CD pipelines, N sets of dashboards/
  alerts/on-call runbooks, N sets of dependency upgrades to keep current.
- **Service boundaries are a data-modeling problem first**: get them wrong and you get chatty
  services that call each other constantly (all the network cost, none of the isolation
  benefit) — this is often worse than the monolith you left.

## Core patterns

- **API gateway**: single entry point for external clients, handles auth, rate limiting,
  routing, and request fan-out so clients don't need to know the internal topology.
- **Service discovery**: services find each other's current network location (DNS-based, or a
  registry like Consul/etcd/Kubernetes Services) instead of hardcoded addresses, since instances
  come and go with scaling/deploys.
- **Circuit breaker**: stop calling a service that's failing/slow, fail fast instead of piling up
  timeouts and exhausting the caller's own thread/connection pool (cascading failure). Standard
  states: closed (normal) → open (fail fast) → half-open (probe recovery).
- **Bulkhead**: isolate resource pools (connection pools, thread pools) per downstream dependency
  so one slow dependency can't starve calls to everything else.
- **Backend-for-frontend (BFF)**: a thin per-client (web/mobile) aggregation service that composes
  calls to multiple backend services, keeping that fan-out logic out of the client.
- **Sidecar/service mesh** (Envoy, Istio, Linkerd): offload cross-cutting concerns (mTLS, retries,
  tracing, load balancing) out of application code into a per-instance proxy.

## Staff-engineer notes

- Draw service boundaries around **bounded contexts** (DDD term: an area with its own consistent
  model and vocabulary — "Order" means something different to Fulfillment vs. Billing), not
  around database tables or org chart boxes. Boundaries drawn along data-ownership lines age
  better than ones drawn along team-reporting lines.
- The single most common regret in real systems: extracting services too early, before the
  domain boundaries have stabilized, so services end up needing frequent breaking-boundary
  changes anyway — coordinated across N repos instead of one PR in a monolith.
- Track a "chattiness" metric per service pair (inter-service calls per user request). A rising
  trend there is the signal a boundary is wrong, well before it becomes a postmortem.
- Own the platform cost honestly in RFCs: N services means N times the baseline operational
  overhead (dashboards, alerts, on-call load, dependency upgrades) even before any new feature
  work — this is the line item people forget to budget for.
