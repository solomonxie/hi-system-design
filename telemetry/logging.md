# Logging

## Structured over free-text

Emit logs as structured data (JSON, or key=value) rather than free-text sentences. A free-text
log needs a regex to query; a structured log is queryable/filterable/aggregable by field
directly in whatever backend ingests it. `level=error msg="payment failed" order_id=123
gateway=stripe reason=card_declined` beats `"Error: payment for order 123 failed via stripe -
card declined"` even though a human reads the second one more easily at a glance — optimize for
the tool doing the reading, since that's what happens at any real volume.

## Log levels, used consistently

- **ERROR**: something failed that needs human attention (paged or ticketed) — an unhandled
  exception, a dependency that's down.
- **WARN**: something unexpected but handled/recovered — a retry succeeded, a fallback kicked in.
- **INFO**: significant business events worth keeping (order placed, user signed up) — the
  backbone of an audit trail.
- **DEBUG**: verbose detail useful in development or active incident investigation, normally
  off/sampled in production because of volume.

The recurring failure mode is everything logged at INFO or ERROR with nothing in between,
making WARN-worthy degradation invisible until it's already an incident.

## What belongs in every log line

A request/trace ID, a timestamp (UTC, consistent format), the service/component name, and enough
context (user/tenant/entity ID) to filter to just this request across a sea of concurrent
traffic. Missing the trace ID is the single most common thing that turns "read the logs" into
"grep and pray" during an incident.

## Sampling and retention

At real scale, logging every line at full volume is not economically viable. Common approach:
log 100% of ERROR/WARN, sample INFO (e.g. 1-10%) or aggregate it into metrics instead, and set a
retention window matched to actual need (hot/searchable for days-to-weeks, cold/archived for
compliance-driven longer windows) rather than "keep everything forever by default," which is
usually just an unbudgeted, growing cost line.

## Staff-engineer notes

- **Never log secrets or PII in plaintext** — API keys, passwords, full card numbers, raw
  personal data. This is a recurring, expensive class of incident (log data usually has much
  laxer access controls and much longer retention than the primary datastore it's describing).
  Redact/mask at the logging layer, not by convention/hope at every call site.
- Log the *decision*, not just the *data*: "rate limit exceeded, rejecting request" is more
  useful during an incident than a log line that only shows the numbers you'd have to
  reverse-engineer the logic from.
- A log line that only makes sense with the surrounding code open is a log line that will be
  useless to whoever's on call at 3am and isn't you. Write the message assuming the reader has
  no other context.
