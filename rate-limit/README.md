# Rate Limiting

Bounding how much traffic a client (user, API key, IP, tenant) can send in a window — protects
capacity, enforces pricing tiers, and contains abuse/bugs (a misbehaving client shouldn't be able
to degrade service for everyone else).

## Algorithms

- **Fixed window counter**: count requests in a fixed clock window (e.g. per-minute), reset at
  the boundary. Simplest to implement, but bursts at window edges: a client can send the full
  limit right before a window ends and again right after, getting 2x the intended rate in a short
  span around the boundary.
- **Sliding window log**: keep a timestamp per request, count how many fall within the trailing
  window on each check. Precise (no boundary burst problem), but memory cost scales with request
  volume per client, which is expensive at high rates.
- **Sliding window counter**: approximates the sliding log cheaply — weight the previous fixed
  window's count by how much of it overlaps the current sliding window, combine with the current
  window's count. Standard practical compromise: nearly as accurate as the log, nearly as cheap
  as the fixed counter.
- **Token bucket**: a bucket holds up to N tokens, refills at a steady rate, each request consumes
  a token (rejected if empty). Naturally allows bursts up to bucket size while enforcing a steady
  average rate — the standard choice when some burstiness is fine (most APIs) since it matches
  real traffic shape better than a hard per-window cap.
- **Leaky bucket**: requests queue and are processed at a fixed output rate regardless of input
  rate (like a bucket with a hole draining at constant speed); this smooths bursts into a
  constant rate rather than allowing them, which is a real functional difference from token
  bucket — appropriate when downstream truly can't handle bursts at all (e.g. protecting a fixed
  processing pipeline), not just "another rate limiter."

## Where to enforce it

- **Client-side**: cooperative only (a well-behaved SDK backs off) — never a security boundary,
  since a client can just not do this; still worth building into your own SDKs to be a good API
  citizen and reduce needless 429s.
- **API gateway / edge**: the standard place for the *coarse* limit (per-API-key, per-IP) —
  rejects abusive traffic before it costs any backend compute at all, which is the whole point of
  putting it as far upstream as possible.
- **Per-service**: finer-grained, resource-specific limits close to the actual constrained
  resource (e.g. a specific expensive endpoint limited tighter than the general API limit) —
  layered on top of the gateway-level limit, not instead of it.
- **Distributed enforcement**: at more than one instance, the counter needs to be shared (Redis
  with `INCR` + `EXPIRE`, or a purpose-built rate-limiting service) — a naive per-instance
  in-memory counter behind a load balancer with N instances silently multiplies the effective
  limit by N, since each instance only sees its own share of traffic.

## What to return

`429 Too Many Requests`, with a `Retry-After` header (and ideally `X-RateLimit-Limit` /
`X-RateLimit-Remaining` / `X-RateLimit-Reset` headers) so well-behaved clients can back off
correctly instead of hammering retries — a rate limiter that doesn't tell the client when to try
again just converts one problem (too much traffic) into another (blind retry storms).

## Staff-engineer notes

- Rate limits are a multi-tenant fairness mechanism as much as an abuse defense — the design
  question in most real systems isn't "what's the global limit" but "how do we stop one noisy
  tenant from degrading service for everyone else sharing the same backend capacity" (per-tenant
  quotas, or fair-queuing rather than a single global limit).
- Distinguish "reject" from "throttle/degrade": for some endpoints, serving a slightly stale
  cached response or shedding a non-critical feature under load is a better user experience than
  a hard 429 — rate limiting and graceful degradation are related but distinct tools, and
  conflating them in a design leads to either unnecessary 429s or missed degradation
  opportunities.
- A rate limiter is itself a piece of shared infrastructure with its own failure mode — decide
  explicitly what happens if the rate-limiting store (Redis) is unreachable: fail open (allow all
  traffic, risking overload) or fail closed (reject all traffic, a self-inflicted outage) — both
  are defensible depending on the system, but it needs to be a decision, not a default nobody
  chose.
