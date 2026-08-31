# Distributed Tracing & OpenTelemetry

## The core idea

A trace is the path of one request across every service/function it touches, represented as a
tree of **spans** — each span is a timed operation (an HTTP call, a DB query, a function) with a
start time, duration, and parent span. The trace is what lets you answer "where did the 2 seconds
go" for a request that crossed 8 services, which no amount of staring at 8 separate services'
logs will reconstruct as reliably.

## Key concepts

- **Trace ID**: one per originating request, propagated through every downstream call (usually
  via an HTTP header — `traceparent` in the W3C Trace Context standard).
- **Span**: one unit of work within the trace; has its own span ID and a parent span ID, forming
  the call tree.
- **Context propagation**: the mechanism that carries the trace ID (and span ID, baggage) across
  process/network boundaries — in-process via thread-local/async-context, cross-process via
  headers (HTTP) or message metadata (queues). This is the part that's easy to silently break —
  any hop that doesn't forward the header (a proxy, a queue without metadata support, a
  fire-and-forget goroutine) truncates the trace there.
- **Sampling**: capturing every trace at high volume is expensive; head-based sampling (decide at
  the start of the trace) is simple but can miss rare slow/error traces; tail-based sampling
  (decide after the trace completes, keeping errors/slow traces preferentially) is more useful
  but requires buffering the whole trace before deciding.

## OpenTelemetry (OTel)

The vendor-neutral standard for generating and exporting all three signal types (traces, metrics,
logs) — an instrumentation API/SDK plus the OTLP wire protocol, decoupled from any specific
backend. Practically:

- Instrument once with the OTel SDK/auto-instrumentation libraries; point the exporter at
  whatever backend you use (Jaeger, Tempo, Honeycomp, Datadog, X-Ray, etc.) — switching backends
  later doesn't mean re-instrumenting the app.
- The **Collector** is a standalone process that receives OTLP data, can batch/filter/sample it,
  and fans it out to one or more backends — the standard place to put sampling policy and
  vendor-specific export logic, out of application code.
- Auto-instrumentation exists for most common frameworks/libraries (HTTP servers/clients, DB
  drivers, gRPC) — this gets you spans for free at service boundaries; manual instrumentation
  (custom spans around business logic) is still needed for anything domain-specific worth seeing
  in a trace.

## Staff-engineer notes

- Standardizing on OpenTelemetry org-wide (rather than a vendor-specific SDK) is the kind of
  platform decision worth making explicitly and early — it's cheap to adopt from day one and
  expensive to retrofit once every service has hand-rolled, vendor-locked instrumentation.
- Trace ID propagation across an async boundary (a message queue, a background job) needs
  deliberate wiring — it doesn't happen automatically the way it does for a synchronous HTTP
  call, and this is exactly the boundary where "why did this become disconnected from the
  original request" incidents come from.
- Tracing answers "which hop was slow"; it doesn't replace logs for "why was that hop slow" (the
  actual error, the actual query plan). Link trace ID into log lines so you can jump from a slow
  span straight to the relevant logs.
