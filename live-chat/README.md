# Live Chat

Real-time, bidirectional message delivery at scale — the same core problem whether it's a
1:1 DM, a group chat, or a live-stream comment feed, differing mainly in fan-out size.

## Delivery transport

- **Polling**: client repeatedly asks "anything new?" on an interval. Simplest, works
  everywhere, but wastes requests when there's nothing new and adds up to interval-length latency
  — rarely the right default for anything marketed as "real-time."
- **Long polling**: client asks, server holds the request open until there's something to send
  (or a timeout), client immediately re-asks. Better latency than plain polling, works over
  plain HTTP (no special infra), but each held-open request ties up a server connection/thread —
  a real scaling constraint at high concurrent-user counts unless the server is built for it
  (async I/O, not one-thread-per-connection).
- **WebSocket**: a single persistent, full-duplex TCP connection after an HTTP upgrade handshake —
  the standard choice for real chat. Low per-message overhead (no repeated HTTP headers), true
  server-push. Cost: the connection is stateful, so it needs a server that holds it open
  (capacity planning is "concurrent connections," not "requests/sec"), and a load balancer that
  supports sticky/persistent connections.
- **Server-Sent Events (SSE)**: one-directional (server → client) persistent HTTP connection —
  simpler than WebSocket when the client never needs to push over the same channel (e.g. it can
  send messages via normal HTTP POST and only needs push for receiving), with the advantage of
  working over plain HTTP/2 without an upgrade handshake.

## Core architecture

- **Connection/gateway layer**: stateful servers holding the open WebSocket connections, one
  user's connection pinned to one server instance — this is why chat services need a way to
  route "deliver a message to user X" to whichever specific gateway instance X is currently
  connected to (a presence registry mapping user → server instance, often in Redis).
- **Message fan-out**: for a 1:1 chat, fan-out is trivial (one recipient); for a group/channel, the
  message needs delivering to every connected member — implemented as a pub/sub layer (Redis
  Pub/Sub, Kafka, or a purpose-built system) that the gateway servers subscribe to per
  active room/channel, so a message published once reaches every gateway with a connected
  member.
- **Message persistence**: chat history needs a durable store independent of the real-time path —
  the real-time delivery and the durable write are usually decoupled (write to the DB, publish
  to the fan-out layer, both from the same request) so a connection drop never means lost
  history, only a missed *live* push (recovered on reconnect via a "give me messages since X"
  catch-up fetch).
- **Presence**: online/offline/typing status — inherently ephemeral, high-write-volume state,
  usually kept in a fast in-memory store (Redis) with TTL-based expiry rather than the primary
  durable database, and often deliberately eventually-consistent (a few seconds of staleness on
  "is this user online" is an acceptable, expected trade for not hammering the primary DB with
  every typing-indicator keystroke).

## Ordering and delivery guarantees

Per-conversation message ordering matters far more than global ordering — a per-conversation
sequence number (or a Lamport-style logical clock) lets clients detect and correctly order
out-of-order delivery without needing a single global order across the whole system. Delivery is
typically at-least-once (network hiccups, reconnects) — clients need to de-duplicate by a
message ID the sender generated (so a retried send doesn't show as two messages) rather than
assume the transport guarantees exactly-once.

## Staff-engineer notes

- Reconnection/catch-up is the part that separates a chat demo from a real product: on
  reconnect, the client needs "give me everything since my last known message ID/timestamp," and
  the server needs that history retained and queryable — don't design the real-time path without
  designing this recovery path alongside it from the start.
- Group size drives real architectural choices: a naive "publish to every member's connection"
  fan-out is fine for a 5-person group chat and a genuine bottleneck for a 100k-member channel or
  a live-stream comment feed — large-fan-out cases usually need a different delivery strategy
  (batching, sampling, a separate high-fan-out broadcast path) than 1:1/small-group chat, so
  don't assume one mechanism covers both without checking the numbers.
- Connection-layer capacity planning is about concurrent open connections and their idle
  keep-alive cost, not request throughput — this is a different sizing exercise than a typical
  stateless HTTP API, and picking a framework/runtime that handles many idle connections cheaply
  (async I/O) matters more here than raw per-request throughput.
