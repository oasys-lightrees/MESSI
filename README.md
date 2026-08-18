# MESSI

**M**essaging, **E**valuation/**S**creening, **S**ystems & **I**ssues — Lightrees' internal
operations and AI workflow platform.

MESSI turns unstructured operational activity (conversations, forms, updates) into
structured records, workflows and decisions, with an AI layer applied only where it
produces measurable operational value.

```
Conversation / Screening → Structured record → Workflow / Project → Issue / Task → AI summary & action
```

This repository currently contains the **system design** for the platform. No application
code has been written yet; the design is the artefact under review.

## Design documents

| # | Document | What it answers |
|---|----------|-----------------|
| 01 | [Overview & Scope](docs/01-overview.md) | What we are building, for whom, what is explicitly out of scope |
| 02 | [Architecture](docs/02-architecture.md) | Components, boundaries, request/data flows, service topology |
| 03 | [Domain Model](docs/03-domain-model.md) | Entities, relationships, lifecycles, the screening→project→issue chain |
| 04 | [Data Model](docs/04-data-model.md) | PostgreSQL schema, indexes, tenancy columns, migrations |
| 05 | [API Design](docs/05-api-design.md) | REST surface, conventions, errors, pagination, idempotency |
| 06 | [AI Layer](docs/06-ai-layer.md) | AI task contracts, grounding, cost control, quality evaluation |
| 07 | [Events & Automation](docs/07-events-and-automation.md) | Outbox, job queue, rules engine, notifications |
| 08 | [Security & Tenancy](docs/08-security-and-tenancy.md) | AuthN/AuthZ, RBAC, isolation, data governance |
| 09 | [Deployment & Operations](docs/09-deployment-and-operations.md) | Docker, AWS topology, CI/CD, observability, runbooks |
| 10 | [Delivery Plan & KPIs](docs/10-delivery-plan.md) | Phase map, KPI scorecard, measurement mechanics, risk register |
| 11 | [Phase 1 — Internal MVP](docs/11-phase-1-mvp.md) | Scope, sequencing, exit criteria — the no-AI baseline phase |
| 12 | [Phase 2 — Automation](docs/12-phase-2-automation.md) | Messenger, AI rollout discipline, exit criteria |
| 13 | [Phase 3 — Productization](docs/13-phase-3-productization.md) | Multi-tenancy, billing, operating-model changes |

Architecture decisions with lasting consequences are recorded in [docs/adr](docs/adr).

## Design principles

1. **Deterministic software does deterministic work.** AI summarizes, classifies and
   extracts. It never owns state transitions, permissions or arithmetic.
2. **Structure is captured once.** A fact entered during screening is never re-typed to
   become a project, an issue or a report.
3. **Organization-aware from day one.** Every row carries an owning organization even
   though Phase 1 has exactly one. Multi-tenancy in Phase 3 is then a configuration
   change, not a migration.
4. **Measure the outcome, not the AI usage.** Every KPI on the proposal's scorecard has a
   named table, event or metric behind it before the feature ships.
5. **Boring until proven otherwise.** One database, one deployable backend, one queue
   mechanism. Extraction into services happens when a measurement demands it.
