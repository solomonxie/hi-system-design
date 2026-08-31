# Sharding (Partitioning)

Splitting data across multiple nodes so no single node has to hold or serve all of it. The
partitioning-key decision is the single choice that determines almost everything downstream —
hot spots, cross-shard query cost, and how painful resharding will be later.

## Partitioning strategies

- **Range-based**: contiguous key ranges per shard (e.g. user IDs 1-1M on shard A, 1M-2M on shard
  B). Makes range queries (and ordered scans) efficient, but is prone to hot spots when access
  pattern correlates with key order (e.g. time-ordered IDs mean all recent writes hit the newest
  shard).
- **Hash-based**: `hash(key) % N` (or, better, consistent hashing — see below) spreads keys
  pseudo-randomly across shards, avoiding the range-based hot-spot problem, at the cost of losing
  efficient range queries (adjacent keys are scattered across shards).
- **Directory-based**: an explicit lookup service maps key → shard. Most flexible (arbitrary,
  reassignable mapping) but adds a dependency and a potential bottleneck/single point of failure
  at the lookup layer unless it's itself replicated and cached.
- **Geo/tenant-based**: partition by a natural business dimension (region, customer/tenant) —
  common in multi-tenant SaaS and systems with data-residency requirements; makes per-tenant
  operations (backup, deletion, compliance) clean at the cost of needing careful handling for any
  cross-tenant query.

## Consistent hashing

The standard fix for the "adding/removing a node reshuffles almost everything" problem with naive
`hash(key) % N`: nodes and keys are placed on a hash ring, and a key belongs to the next node
clockwise from it. Adding/removing a node only remaps the keys between it and its neighbor on the
ring — not the whole keyspace. Virtual nodes (each physical node owns many points on the ring)
smooth out load distribution that a single point per node wouldn't. This is the mechanism behind
DynamoDB, Cassandra, and most distributed caches' data placement.

## The choice of shard key

Pick a key that (a) matches your actual access pattern — most queries should be satisfiable
within one shard, since cross-shard queries mean fan-out and merge at the application layer — and
(b) doesn't create a hot shard (a single tenant/key that dwarfs the others in traffic or data
volume). Getting this wrong is expensive to fix later because it usually means a live
resharding/migration, not a config change.

## Resharding

Splitting or rebalancing shards as data grows is an operational project, not a flag flip: it
typically means dual-writing to old and new shard layouts during a migration window, backfilling
historical data, verifying consistency, then cutting reads over — see `migration/` for the
general pattern. Systems that anticipate this (consistent hashing, or a directory layer that can
be repointed) pay much less for it than ones that hardcoded `% N` sharding.

## Staff-engineer notes

- The shard key decision is effectively permanent once there's real data and traffic on it —
  treat it with the scrutiny of a schema decision, not a config value, and model expected growth
  and hot-key risk before committing.
- A cross-shard transaction or cross-shard query is a strong signal the shard key doesn't match
  the access pattern — either the key needs to change, or that operation needs to be redesigned
  (denormalize, precompute, accept eventual consistency) rather than bolted on with distributed
  transactions (see `distributed-transactions.md`) as a patch.
- Uneven shard load ("hot shard") is usually a monitoring gap before it's an incident — track
  per-shard request rate and data volume, not just aggregate cluster metrics, or the imbalance is
  invisible until one shard falls over.
