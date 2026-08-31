# Architecture Styles

Four ways to draw the boxes-and-arrows diagram for the same product. None of these is "correct"
in the abstract — each trades org-scaling and deploy-independence against operational and
cognitive overhead. The failure mode on both sides is common: staying a monolith past the point
your team structure needs independent deploys, or splitting into microservices before you have
the traffic, team size, or platform investment to run them.

## Quick comparison

| | Monolith | Microservices | Event-driven | Serverless |
|---|---|---|---|---|
| Unit of deploy | whole app | per service | per service/handler | per function |
| Scaling granularity | coarse (whole app) | fine (per service) | fine (per consumer group) | automatic, per invocation |
| Team ownership | shared | one team per service | one team per producer/consumer | one team per function/service |
| Failure blast radius | large (one bug, one crash) | contained to a service | contained, but async debugging is harder | contained, but cold-start/limits bite |
| Operational cost | lowest | highest (N services to run) | medium-high (broker to run) | shifted to the cloud vendor |
| Latency | in-process calls | network hop per call | usually async, not request/response | cold starts add tail latency |
| Local dev / debugging | easiest | hardest (many services to stand up) | hard (causality is spread across events) | easy per function, hard end-to-end |

## The real driver: Conway's Law

Architecture tends to mirror communication structure whether you plan it or not. Microservices
solve an org problem (N teams need to deploy independently without blocking each other) as much
as a technical one. If you have one team, splitting into 10 services usually just adds latency
and ops burden without buying anything — you can't parallelize deploys you were never blocked on.

## Staff-engineer notes

- The default recommendation for a new product is a monolith with clean internal module
  boundaries — it's the cheapest to build, the easiest to refactor, and the boundaries you draw
  early are usually wrong anyway. Extract a service once a boundary has proven stable under real
  usage and a team needs to own its own deploy cadence.
- "Modular monolith" (strict internal module boundaries, single deploy) is underused as a middle
  ground — it gets most of microservices' maintainability benefit without the distributed-systems
  tax (network partitions, partial failure, distributed tracing, service mesh).
- Migrations between styles are rarely "big bang." The strangler fig pattern (new functionality
  built as a service, old functionality migrated incrementally, monolith proxies/routes to the
  new service until it's empty) is the standard way to decompose a monolith without a rewrite
  freeze.
- Event-driven and microservices aren't mutually exclusive — most real microservice systems use
  synchronous calls (REST/gRPC) for read paths and events for cross-service side effects
  (`OrderPlaced` triggers inventory, billing, notifications independently). Mixing the two well is
  the actual skill; see `message-queue/` for the mechanics.
