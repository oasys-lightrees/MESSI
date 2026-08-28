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
| [0007](0007-followup-cycles-generalize-screening.md) | Follow-up cycles generalize screening; one engine, many modules | Accepted |
| [0008](0008-commitment-driven-cadence.md) | Cadence derives from what the player promised, not a fixed calendar | Accepted |
| [0009](0009-attribution-computed-narrative-written.md) | Bottleneck attribution is computed; the model only names and narrates | Accepted |
| [0010](0010-question-quality-gated-at-authoring.md) | Question quality is gated at authoring, not policed at answering | Accepted |
