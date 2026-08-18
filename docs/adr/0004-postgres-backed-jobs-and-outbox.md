# ADR-0004 — Postgres for jobs and the outbox; no message broker

**Status:** Accepted · **Date:** 2026-08-18

## Context

MESSI needs a transactional outbox, a background job queue and scheduled work. The
conventional answers are SQS, RabbitMQ, Kafka, or Redis-backed queues.

The pilot's workload is on the order of thousands of jobs per day.

## Decision

PostgreSQL for all three, using `SELECT … FOR UPDATE SKIP LOCKED`:

- `outbox` — written in the same transaction as the domain change, dispatched by a poller.
- `jobs` — general work queue with `dedupe_key`, `run_at`, attempts and stale-lock recovery.
- Scheduling via a leader-elected scheduler holding a Postgres advisory lock.

## Consequences

**Good**
- **The outbox is atomic with the business write.** With an external broker this requires
  two-phase commit or an outbox in Postgres anyway — so the broker would be added
  machinery, not a replacement for this table.
- Zero additional infrastructure to provision, monitor, secure and pay for.
- Jobs are queryable with SQL. "Why did this notification not fire" is a `SELECT`, not a
  broker console expedition.
- Backup and restore covers the queue state along with the data. No split-brain between a
  restored database and a broker holding stale messages.
- Local development needs one container.
- Dead-letter handling and replay are ordinary rows and ordinary endpoints.

**Bad, accepted**
- Polling adds latency (~1 s) and constant low-level database load. Both are negligible
  here; `LISTEN/NOTIFY` can reduce latency later without changing the correctness model.
- Queue tables grow and need pruning — scheduled in [doc 04](../04-data-model.md) §4.11.
- Long-running jobs hold locks; mitigated by the 5-minute stale-lock reaper and by keeping
  jobs short.
- This does not scale to six-figure jobs per second. Nothing in the three-phase plan
  approaches that.

**Revisit when:** sustained throughput exceeds ~100 jobs/second, or a consumer outside
MESSI needs the event stream. At that point the outbox becomes the source that feeds a
broker — the outbox table itself still stays.
