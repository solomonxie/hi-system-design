# System Design Notes

Reference notes on system design — the core concepts, and the patterns a staff-level engineer
actually reaches for day to day (architecture reviews, incident retros, capacity planning, RFC
writing). Written for both interview prep and real work; every topic tries to cover not just
"how does X work" but "when do you actually reach for X, and what goes wrong when you do."

## Scope

- **Concepts over trivia**: trade-offs and failure modes, not just definitions.
- **Staff-engineer angle**: each topic calls out the parts that show up in RFCs, design reviews,
  and postmortems — not just what gets asked in interviews.
- **Opinionated where it matters**: "it depends" is true but not useful on its own; notes state a
  default and the conditions that would change it.

## Topics

- **architecture styles**: monolith, microservices, event-driven, serverless — when each earns
  its complexity, and the migration paths between them.
- **telemetry**: logging, metrics, tracing, OpenTelemetry, Prometheus/Grafana, the ELK stack —
  observability as a system property, not an afterthought.
- **distributed systems**: caching, consensus, consistency models, sharding, distributed
  transactions.
- **frontend architecture**: rendering strategies (SSR/CSR/SSG/ISR), cross-site security
  (CORS/CSRF/XSS, cookies).
- **video streaming**: adaptive bitrate, CDN delivery, live vs. VOD.
- **live chat**: real-time delivery, presence, fan-out at scale.
- **authentication**: session vs. token, OAuth2/OIDC, SSO, MFA.
- **rate limiting**: algorithms, where to enforce them, multi-tenant fairness.
- **migration**: schema/data migrations at scale, zero-downtime cutover patterns.
- **message queue**: delivery guarantees, ordering, backpressure, dead-letter handling.
- **networking**: VPCs, subnets, load balancing, DNS, service discovery.

## How these notes are organized

Each topic folder opens with a short overview, then one file per subtopic going deeper. Notes
lean on comparison tables and concrete numbers over prose where possible — the kind of thing
that's actually useful mid-design-review, not just on a first read.
