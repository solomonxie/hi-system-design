# Distributed Transactions

How to coordinate a write that spans more than one datastore or service without a single
database's ACID transaction to fall back on. This comes up the moment you have more than one
service each owning its own database (see `architecture-styles/microservices/`) — "update the
order and decrement inventory" is no longer one transaction, it's two systems that both need to
end up consistent.

## Two-phase commit (2PC)

A coordinator asks every participant to **prepare** (do the work, lock resources, but don't
commit) and confirm they can commit; only once all participants confirm does the coordinator send
**commit** to everyone. Strongly consistent, but:

- Blocking: participants hold locks from prepare until commit — a slow or crashed coordinator
  stalls every participant, holding their resources locked the whole time.
- The coordinator is a single point of failure for the whole transaction; if it crashes after
  some participants committed and before others heard, participants are left in an ambiguous
  state until it recovers.

Mostly seen today inside a single database engine's internals or between tightly-coupled systems
in the same trust/failure domain (e.g. XA transactions across two databases in the same
datacenter) — rarely a good fit across independently-deployed microservices because of the
blocking/coupling cost above.

## Sagas

The standard pattern for cross-service consistency without 2PC: break the transaction into a
sequence of local transactions, each with a defined **compensating action** that undoes it if a
later step fails. "Reserve inventory" compensates with "release inventory"; "charge card"
compensates with "refund." The system reaches consistency eventually, through forward progress or
compensation, not through a single atomic commit.

- **Choreography**: each service publishes an event on completing its step; the next service
  reacts to it (see `architecture-styles/event-driven/`). No central coordinator, but the overall
  flow is implicit — spread across every service's event handlers, which makes it harder to see
  the whole flow in one place, or to add a new step without touching multiple services.
- **Orchestration**: a central saga orchestrator explicitly calls each step and its compensation
  in order. The flow is visible in one place (easier to reason about, easier to add a step), at
  the cost of that orchestrator being a new component with its own availability/ownership
  concerns.

Sagas are not atomic — there's a real window where the system is in a partially-completed state,
visible to anyone reading it during that window. Whether that's acceptable is a product decision,
not just a technical one (e.g. an order showing "processing" is often fine; a bank balance
briefly showing a debit before the linked credit lands usually isn't).

## The outbox pattern

Solves a narrower but very common problem: writing to your own DB and publishing an event about
that write need to be atomic, but a DB transaction and a broker publish aren't naturally one unit.
Fix: write the event to an "outbox" table in the *same* local DB transaction as the business
write (this part is just a normal ACID transaction), then a separate relay process
(polling the table, or reading its DB's change log via CDC) publishes it to the broker
asynchronously and marks it sent. Guarantees the event is eventually published if and only if the
write committed — no "wrote to DB but the publish failed" gap.

## Staff-engineer notes

- Default to sagas + outbox for cross-service consistency; reach for 2PC only within a tightly
  coupled, same-failure-domain boundary where its blocking behavior is acceptable — this is the
  standard shape of real systems, and interview answers that reach straight for 2PC across
  microservices are usually flagged for exactly this reason.
- Every saga step needs its compensating action designed and tested *as carefully as the forward
  path* — compensations are the part that only runs during failure, which means they're the part
  least exercised in normal operation and most likely to have a bug nobody's found yet.
- Idempotency is not optional for saga steps or their compensations: retries (from timeouts, from
  at-least-once delivery) mean any step or compensation can run more than once, and it must
  produce the same end state each time (e.g. "release inventory" should be safe to call twice,
  not double-release).
