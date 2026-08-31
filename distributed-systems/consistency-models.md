# Consistency Models & CAP/PACELC

## CAP, stated precisely (and its usual misreading)

Given a network **P**artition, a system must choose between **C**onsistency (every read sees the
latest write) and **A**vailability (every request gets a response). You do not choose "CA" —
partitions happen (a network is not perfectly reliable), so the real choice is CP or AP *during a
partition specifically*. Outside of a partition, a well-built system can be both consistent and
available; CAP is a statement about partition behavior, not a permanent 2-of-3 trade you live in
at all times.

## PACELC: the more complete framing

Extends CAP with the case that actually dominates day-to-day operation: **if Partitioned**,
choose **A**vailability or **C**onsistency; **Else** (normal operation, no partition), choose
**L**atency or **C**onsistency. Even with no partition, strong consistency (waiting for a
majority of replicas to confirm a write) costs latency compared to acknowledging as soon as one
node has it. This is the trade-off you're actually making on the vast majority of requests, since
partitions are (hopefully) rare.

## The consistency spectrum

Not binary — a range of guarantees, strongest to weakest, each with a real cost:

- **Linearizability (strict consistency)**: every operation appears to happen instantaneously at
  some point between its start and end; any read after a write completes sees that write,
  globally. Strongest, most expensive (effectively requires consensus-level coordination — see
  `consensus.md`).
- **Sequential consistency**: all nodes see operations in the same order, but that order need not
  match real-time — weaker than linearizability, still strong enough for many correctness
  arguments.
- **Causal consistency**: operations that are causally related (B read a value A wrote) are seen
  in that order by everyone; unrelated operations can be seen in different orders on different
  nodes. Cheaper than sequential, and matches human intuition about "cause before effect" well
  enough for many collaborative apps (comments, chat — see `live-chat/`).
- **Read-your-writes**: a specific, practically important guarantee — a client always sees its
  own prior writes, even if it doesn't see others' writes promptly. Commonly implemented by
  routing a user's own reads to the primary (or a replica known to be caught up) right after they
  write.
- **Monotonic reads**: once a client has seen a value, it never sees an older value on a
  subsequent read (no "time travel backwards" from replica lag making a later read look stale
  relative to an earlier one).
- **Eventual consistency**: the weakest useful guarantee — if writes stop, all replicas
  eventually converge to the same value, with no bound on how long "eventually" takes. Cheapest,
  most available; correct for data where staleness is tolerable (a like count, a search index).

## Staff-engineer notes

- Name the specific guarantee a feature needs, not just "consistent" or "eventual" as a whole
  system property — a shopping cart usually needs read-your-writes (you should see the item you
  just added) but can tolerate eventual consistency for "customers who bought this also bought."
  Mixing per-feature is normal and correct, not a compromise.
- Replica lag is the everyday, low-drama version of the "eventual" in eventual consistency — a
  read routed to a replica milliseconds behind the primary can return a stale value with no
  partition or failure involved at all. If a design reads from replicas for scale, decide
  explicitly which reads can tolerate that lag and which must go to the primary.
- When a design doc says "eventually consistent," push for a number: eventually consistent within
  what bound, under normal load, and what happens to correctness if that bound is exceeded during
  a backlog/incident. An unbounded "eventually" is not a spec.
