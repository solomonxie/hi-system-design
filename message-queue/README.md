# Message Queues

The infrastructure underneath `architecture-styles/event-driven/` — how messages actually get
buffered, ordered, and delivered between producers and consumers. Same underlying idea across
every implementation; the differences that matter are in delivery guarantees, ordering scope, and
whether messages are consumed-once (queue) or replayable (log).

## Queue vs. log, the real distinction

- **Queue model** (RabbitMQ, SQS, ActiveMQ): a message is delivered to (typically) one consumer
  and then removed/acked — once consumed, it's gone. Natural fit for work distribution (task
  queues, job processing) where each unit of work should be done exactly by one worker.
- **Log model** (Kafka, Kinesis, Pulsar): messages are appended to a durable, ordered log and
  *retained* (for a configured window, not removed on read); consumers track their own read
  position (offset) and can replay from any prior offset. Natural fit for event streaming where
  multiple independent consumers each need the full stream, and replay (rebuilding a read model,
  recovering from a bug by reprocessing) is a real, expected operation.

## Delivery guarantees

- **At-most-once**: message might be lost, never delivered twice. Rarely what you actually want —
  usually the result of not handling failure explicitly (fire-and-forget) rather than a deliberate
  choice.
- **At-least-once**: message is guaranteed delivered, but might be delivered more than once (a
  consumer processes it, crashes before acking, the broker redelivers to another consumer). The
  practical default for most systems — cheap to guarantee, but pushes idempotency onto the
  consumer as a hard requirement (see `distributed-systems/README.md`).
- **Exactly-once**: the message is processed exactly once, no duplicates, no loss. Achievable
  end-to-end only with real cost/constraints (Kafka's transactional/idempotent producer +
  consumer within Kafka-to-Kafka pipelines; generally not achievable for free across an arbitrary
  broker + arbitrary external side effect) — treat "we need exactly-once" claims in a design with
  scrutiny; "at-least-once + idempotent consumer" achieves the same practical outcome far more
  cheaply and is the standard real answer.

## Ordering

Most brokers guarantee order only within a **partition** (Kafka) or a single queue (not across
multiple consumers pulling from it concurrently) — global ordering across a whole topic/queue
either isn't guaranteed or requires giving up parallelism entirely (a single consumer, no
concurrency). The standard pattern: partition by a key that needs ordering relative to itself
(e.g. all events for one `order_id` in the same partition, guaranteeing that order's events are
processed in order) while allowing full parallelism across different keys.

## Backpressure

What happens when consumers can't keep up with producers — the queue/log absorbs the difference
up to its retention/storage limit, buying time, but isn't infinite. Real handling means: consumer
lag as a monitored metric (not discovered when storage fills up), autoscaling consumers to match
producer rate, and a defined behavior for sustained overload (shed low-priority messages, apply
backpressure upstream to producers, or accept growing latency within a bounded window) rather than
an unbounded queue silently growing until something falls over.

## Dead-letter queues (DLQ)

A message that fails processing repeatedly (a poison message — malformed data, a bug triggered
by this specific input) shouldn't block the rest of the queue behind it or retry forever. Standard
handling: after N failed attempts, move it to a separate DLQ, alert, and let a human or a
separate remediation path deal with it — a queue without this either blocks on the first poison
message or silently drops failures, both worse than an explicit DLQ.

## Staff-engineer notes

- Choosing queue vs. log is a real architectural decision, not interchangeable defaults: reach
  for a log (Kafka-style) when replay, multiple independent consumer groups, or event-sourcing-
  style durability matters; reach for a queue (SQS/RabbitMQ-style) for straightforward work
  distribution where "processed once by someone" is the actual requirement and long retention/
  replay isn't needed — a log used purely as a work queue pays retention/complexity cost for a
  capability you're not using.
- Consumer lag is one of the highest-leverage metrics to alert on in any queue-based system — it's
  the earliest, clearest signal that a downstream consumer is falling behind, well before it
  manifests as user-visible staleness or a filled-up queue.
- "We'll just make everything idempotent" is easy to say and easy to get subtly wrong in
  practice — an idempotency key needs to be genuinely unique per logical operation (not per
  retry attempt) and the dedup check needs to be atomic with the side effect it's guarding,
  usually via a unique constraint or a dedup table checked in the same transaction as the write.
