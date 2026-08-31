# Event-Driven Architecture

Services communicate by producing and consuming events (facts about something that happened —
`OrderPlaced`, `PaymentFailed`) through a broker, rather than calling each other directly.
Decouples producers from consumers in both time and topology: a producer doesn't know or care
who (if anyone) is listening.

## Why reach for it

- **Temporal decoupling**: the consumer doesn't need to be up when the event is produced — the
  broker holds it. This absorbs traffic spikes (the queue depth grows instead of the consumer
  falling over) and survives consumer restarts/deploys without dropping work.
- **Fan-out without producer changes**: adding a third consumer to `OrderPlaced` (say, a new
  fraud-check service) requires zero changes to the order service — it's already just publishing
  an event, unlike a direct call it would've had to add.
- **Natural fit for cross-service side effects**: "when an order is placed, update inventory,
  charge payment, send a confirmation email" is inherently a fan-out, not a single synchronous
  call chain.

## What it costs

- **No more request/response**: the producer doesn't get an answer back, and often doesn't know
  if/when a consumer processed the event. If the caller needs a result, event-driven is the
  wrong shape for that part of the request (use a synchronous call, or a request/reply pattern
  over the broker with a correlation ID).
- **Ordering and idempotency become the app's problem**: most brokers guarantee order only within
  a partition/shard, and delivery is usually at-least-once, so consumers must handle
  out-of-order and duplicate events. See `message-queue/`.
- **Debugging causality is hard**: "why did X happen" now means chasing an event through a
  broker and possibly several hops of re-publishing — this is where distributed tracing with
  event/message context propagation (not just RPC spans) earns its keep.
- **Eventual consistency, always**: a consumer's view of the world lags the producer's by however
  long processing + delivery takes. This has to be an explicit, communicated product decision,
  not something a user reports as a bug.

## Core patterns

- **Event notification** (thin event, "OrderPlaced, id=123") vs. **event-carried state transfer**
  (fat event, full order payload included) — thin events mean consumers call back for details
  (extra coupling, extra load); fat events risk staleness and schema bloat. Most real systems
  land on a middle ground: enough payload for the common case, an ID for anything else.
- **Event sourcing**: the event log is the source of truth (current state is a fold/replay over
  events), not a side effect of writing to a table. Powerful for audit trails and time travel,
  expensive in complexity — don't reach for this by default.
- **CQRS**: separate write model (commands, normalized) from read model (queries, denormalized,
  often rebuilt from the event stream). Common pairing with event sourcing but usable
  independently.
- **Outbox pattern**: write the event to an "outbox" table in the same DB transaction as the
  business write, then a separate relay publishes it to the broker — avoids the classic bug of
  "DB write succeeded, event publish failed" (or the reverse) losing consistency between the two.

## Staff-engineer notes

- The event schema is a public contract the moment a second team consumes it — version it
  deliberately (additive fields only, or explicit versioned topics) the same way you'd version a
  public API. A silent breaking change to an event shape is a multi-team incident.
- Push back on "let's just publish everything as events" as a default — synchronous calls are
  simpler to reason about and debug; use events specifically where temporal decoupling or
  fan-out is the actual requirement, not as a house style.
- A dead-letter queue is not optional in production — decide up front what happens to an event
  that fails processing N times (alert + park it, or a defined retry/backoff policy), or it
  becomes silent data loss discovered months later.
