# Caching

Trading staleness for speed by keeping a copy of data somewhere cheaper/closer to read than the
source of truth. The hard part is almost never "where do I put a cache" — it's invalidation:
knowing when the cached copy is wrong and getting rid of it before something acts on stale data.

## Where caches live (closest to farthest from the request)

- **Client-side**: browser cache, mobile app local storage. Zero network cost when it hits, but
  invalidation is effectively best-effort (you can set a TTL, you can't reliably push an
  invalidation to every client instance).
- **CDN / edge**: cached at points-of-presence close to the user, typically for static assets or
  cacheable API responses (see `video-streaming/`). Invalidation is a purge API call, with
  propagation delay across edge nodes.
- **Application-level (in-process)**: an in-memory map/LRU inside the service process. Fastest
  possible hit, but not shared across instances — each replica has its own cold cache after a
  deploy, and invalidating one instance doesn't invalidate the others.
- **Distributed cache** (Redis, Memcached): shared across all instances of a service, one place
  to invalidate. Adds a network hop (still far cheaper than the origin datastore) and its own
  availability/failure mode to design around.

## Cache strategies

- **Cache-aside (lazy loading)**: app checks cache, on miss reads from the source and populates
  the cache. Simplest, most common; cache only ever contains what's actually been requested.
  Risk: a cache-down event sends full traffic straight to the origin (thundering herd) unless
  guarded (see below).
- **Write-through**: writes go to the cache and the source together (synchronously) — cache is
  always consistent with the source, at the cost of every write paying the cache-write latency
  too.
- **Write-behind (write-back)**: writes go to the cache immediately, persisted to the source
  asynchronously — fast writes, but a cache failure before the async flush loses data. Rarely
  worth the risk outside of specific high-write-throughput, loss-tolerant use cases.
- **Read-through**: the cache itself (not the app) knows how to load from the source on a miss —
  same idea as cache-aside, with the loading logic owned by the cache layer instead of every
  caller.

## Invalidation

- **TTL (time-to-live)**: simplest and most common — accept up to TTL-length staleness, no
  explicit invalidation logic needed. Picking the TTL is a product decision (how stale is
  acceptable) as much as a technical one.
- **Explicit invalidation on write**: the write path deletes/updates the cache entry when the
  source changes. More precise (no staleness window) but couples every write path to
  remembering to invalidate — a missed invalidation path is a silent, hard-to-notice bug class.
- **Cache stampede / thundering herd**: when a hot key expires, many concurrent requests all miss
  simultaneously and all hit the origin at once. Mitigations: lock/single-flight (only one
  request repopulates, others wait), staggered/jittered TTLs so hot keys don't all expire
  together, or serve stale-while-revalidate (return the stale value immediately, refresh in the
  background).

## Staff-engineer notes

- "There are only two hard things in computer science: cache invalidation and naming things" is
  a joke with a real point — budget real design time for invalidation, not just for picking a
  cache technology. Most cache-related incidents are invalidation bugs, not cache-technology
  failures.
- A cache being unavailable should degrade the system (slower, more origin load) not break it —
  never make cache availability a hard dependency for correctness; if a cache miss can produce a
  wrong answer rather than just a slow one, that's a design smell.
- Cache hit rate is a metric worth alerting on, not just tracking — a silent drop in hit rate
  (a deploy that changed cache keys, an expired-and-never-refilled cache) shows up as origin
  load and latency, and is much faster to diagnose from the hit-rate graph directly.
