# Distributed Systems

The hard problems that show up the moment state or computation spans more than one machine:
what happens on a network partition, how do you agree on anything, how do you make something
faster without making it wrong. Every topic here is really the same underlying fact stated
differently — **a network call can be slow, can fail, and you cannot tell those two apart from
the caller's side** — and every pattern (retries, consensus, replication, sharding) exists to
make that fact survivable.

## In this folder

- `caching.md` — trading staleness for speed, and the invalidation problem that's harder than
  the caching itself.
- `consensus.md` — how a set of nodes agree on one value despite failures (Raft, Paxos) and what
  it's actually used for underneath (leader election, config stores, distributed locks).
- `consistency-models.md` — the spectrum from strict/linearizable to eventual, and CAP/PACELC as
  the framing for what you're actually trading off.
- `sharding.md` — splitting data across nodes for scale, the partitioning-key decision that
  determines everything else, and resharding as an operational reality.
- `distributed-transactions.md` — 2PC, sagas, the outbox pattern — coordinating a write across
  more than one datastore/service without silent inconsistency.

## Staff-engineer notes

- Every one of these topics is a spectrum, not a binary choice — "eventual consistency" isn't
  one thing, it's a family of guarantees (read-your-writes, monotonic reads, bounded staleness)
  with very different implementation cost and very different user-facing behavior. Pin down which
  specific guarantee a design actually needs before picking a mechanism.
- The honest question in most real designs isn't "CP or AP" in the abstract — it's "which parts
  of this system need strong consistency (payments, inventory decrement) and which can tolerate
  staleness (a view count, a recommendation feed)." Mixed-consistency systems are the norm, not
  the exception.
- Distributed-systems bugs are disproportionately about the failure paths nobody tested: partial
  writes, retried-but-actually-succeeded requests, clock skew. Idempotency (a retried request has
  the same effect as one request) is the single highest-leverage property to design in from the
  start, because so much else here (retries, at-least-once delivery, sagas) depends on it holding.
