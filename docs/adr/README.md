# Architecture Decision Records

Decisions with consequences that outlive the person who made them. Format: context →
decision → consequences, including the consequences we dislike.

| ADR | Decision | Status |
|---|---|---|
| [0001](0001-modular-monolith-in-rust.md) | Modular monolith in Rust; worker is the same binary | Accepted |
| [0002](0002-org-scoped-single-database-multitenancy.md) | Org-scoped single database with RLS from day one | Accepted |
| [0003](0003-separate-python-ai-service.md) | AI in a separate Python service with no data access | Accepted |
| [0004](0004-postgres-backed-jobs-and-outbox.md) | Postgres for jobs and the outbox; no broker | Accepted |
| [0005](0005-ai-proposes-humans-dispose.md) | AI proposes, humans dispose | Accepted |
| [0006](0006-deterministic-health-ai-narrative.md) | Deterministic health metrics, AI narrative only | Accepted |
