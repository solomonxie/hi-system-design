# Migration

Changing a schema, a datastore, or a running system's behavior while it's live and serving
traffic — the category of work that's disproportionately represented in real staff-engineer
project work, and disproportionately absent from interview prep, despite being where a huge
share of real incidents come from.

## Schema migrations, zero-downtime

The core problem: old code and new code both run simultaneously during a rolling deploy — a
migration that isn't compatible with both generations of code for that window causes errors or
data loss during the deploy itself, not after it.

- **Expand/contract (parallel change)**: the standard safe pattern, in explicit phases —
  (1) *expand*: add the new column/table, nullable or defaulted, deploy code that writes to
  both old and new; (2) *backfill*: populate the new field for existing rows; (3) *migrate reads*:
  deploy code that reads from the new field, still writing both; (4) *contract*: once nothing
  reads the old field, stop writing it and drop it. Every phase is independently deployable and
  reversible — the property that actually makes it zero-downtime, since you can pause or roll
  back at any phase without the system being in a broken state.
- **Never combine an additive and a destructive change in one deploy** — dropping/renaming a
  column in the same release that stops using it removes your rollback path the moment something
  else in that release needs reverting.
- **Backfills at scale** need their own care: batched (not one giant transaction — locks, replica
  lag, and rollback-on-failure all get worse with transaction size), rate-limited (a backfill
  competing with live traffic for the same DB capacity is a self-inflicted incident), and
  idempotent/resumable (a backfill job that dies partway through should be safe to resume, not
  restart from scratch or double-apply).

## Data store migrations

Moving from one database/technology to another (e.g. monolith's DB to a per-service DB during a
microservices extraction, or a vendor migration) — the expand/contract idea generalizes:

- **Dual-write**: application writes to both old and new stores during the transition. Real risk:
  the two writes aren't atomic (see `distributed-systems/distributed-transactions.md`) — a
  partial failure leaves them inconsistent, so this needs an explicit reconciliation/verification
  step, not just "write to both and hope."
- **Change Data Capture (CDC)**: rather than dual-writing from the application, replicate from the
  old store's write-ahead/binlog into the new store continuously — keeps application code
  simpler (single write path) and typically has better consistency properties than
  application-level dual-write, at the cost of needing CDC tooling (Debezium is the common OSS
  choice) in the pipeline.
- **Cutover**: once the new store is verified caught-up and consistent, switch reads over
  (often gradually — a percentage of read traffic, or specific read paths first), keep the old
  store as a fallback for a defined rollback window, then decommission it.

## Staff-engineer notes

- Every migration plan needs an explicit rollback plan for every phase, written down before
  starting — "we'll figure out rollback if something goes wrong" discovered mid-incident, with
  data already partially migrated, is a materially worse position than the same discovery made
  during planning.
- Verification (checksums, row counts, sampled diffs between old and new) is not optional
  overhead — it's the only way to know a migration actually succeeded rather than merely
  completed without throwing an exception. Silent data corruption during migration is a genuinely
  common, genuinely hard-to-detect-after-the-fact failure mode.
- Communicate the migration's blast radius and timeline to on-call/affected teams before starting,
  not after something breaks — the value of expand/contract's reversibility is largely wasted if
  nobody knows the migration is in flight when a related alert fires.
