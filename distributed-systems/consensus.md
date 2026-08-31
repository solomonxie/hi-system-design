# Consensus

How a set of nodes agree on a single value despite some of them being slow, crashed, or
partitioned from the rest — the foundation underneath leader election, distributed locks, and
strongly consistent config/coordination stores. You very rarely implement a consensus algorithm
yourself; the leverage is knowing what it guarantees so you can use (or correctly avoid) the
systems built on it.

## The problem, precisely

N nodes each propose a value; consensus ensures they all agree on the *same* one, even if some
nodes fail, as long as a majority (quorum) are up and can talk to each other. This requires
tolerating the fact that a non-responding node might be dead or might just be slow/partitioned —
and from the outside, at any given moment, you cannot tell which.

## Paxos vs. Raft

- **Paxos**: the original, formally proven algorithm — famously correct, famously hard to
  understand and implement correctly from the paper alone. Most real systems that say "Paxos"
  actually run Multi-Paxos (a practical variant with a stable leader) or one of several
  optimized/simplified descendants.
- **Raft**: designed explicitly for understandability, decomposed into leader election, log
  replication, and safety as separate, easier-to-reason-about pieces. This is why most newer
  systems (etcd, Consul, CockroachDB's range layer) implement Raft rather than Paxos — same
  guarantees, much easier to implement and to audit.

## What it takes to make progress

A **quorum** — typically a strict majority ((N/2)+1 nodes) — must be reachable and agree.
This is why odd numbers of nodes (3, 5) are standard: 3 nodes tolerates 1 failure with a
majority of 2 still reachable; 5 tolerates 2. Adding nodes past what your failure tolerance
needs adds latency (more nodes to hear back from) without adding safety.

## Where you actually meet this in practice

- **Leader election**: picking one node to be "in charge" of a shard/partition/resource,
  survivable across that node's failure (Kubernetes uses etcd/Raft under the hood for exactly
  this).
- **Distributed locks / coordination**: ZooKeeper, etcd, Consul — a strongly consistent store
  other services use to coordinate (leader election, service discovery, distributed config)
  precisely because it's built on consensus and gives linearizable reads/writes.
- **Consistent replicated logs**: the mechanism inside Raft-based databases that keeps every
  replica's write-ahead log identical and in the same order, which is what makes strongly
  consistent reads from any replica safe.

## Staff-engineer notes

- Consensus is expensive (every write needs a round trip to a majority of nodes) — reach for a
  consensus-backed store specifically for the coordination/metadata layer (who's the leader,
  what's the current config), not as the primary datastore for high-volume application data.
  Using etcd as a general-purpose database is a well-known way to have a bad time at scale.
- A network partition doesn't "pause" consensus — the minority side simply cannot make progress
  (by design, this is what prevents split-brain). If your system depends on a consensus store and
  that store's quorum is unreachable, understand explicitly what your system does: block, or
  fail open with degraded guarantees. Decide that on purpose, not by accident.
- Don't build a bespoke leader-election/coordination mechanism when a battle-tested one (etcd,
  ZooKeeper, a cloud-managed equivalent) is available — the edge cases (split-brain during
  partition, safe leader handoff) are exactly the part that's easy to get subtly wrong and hard
  to test.
