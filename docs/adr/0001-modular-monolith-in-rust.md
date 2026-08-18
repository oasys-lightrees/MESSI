# ADR-0001 — Modular monolith in Rust; worker is the same binary

**Status:** Accepted · **Date:** 2026-08-18

## Context

The proposal fixes the backend as Rust / Actix Web. It does not fix the number of
deployables. The obvious alternatives were one service per domain module (screening,
projects, issues, messaging) or a single service.

Pilot scale is 25 users. The domain is highly interconnected: converting a screening into a
project touches customers, projects, issues, events and notifications in one operation that
must be atomic.

## Decision

One backend codebase, one deployable image, compiled into a Cargo workspace with enforced
internal boundaries:

- `domain` — entities and state machines, no I/O, no framework dependencies
- `app` — use cases, transaction boundaries, ports
- `infra` — sqlx, S3, AI client, mail
- `api` — Actix Web
- `worker` — background entrypoint, **same binary as `api`**, selected by argv
- `contracts` — shared DTO/event schemas

Modules communicate through domain events and typed service interfaces, never by querying
another module's tables. CI asserts the dependency direction (`domain` must have no
`sqlx`/`actix`/`tokio` in its tree).

## Consequences

**Good**
- The conversion chain is one database transaction. No sagas, no compensating actions, no
  eventual consistency in the feature that is the product's core claim.
- One deployment pipeline, one set of migrations, one place to trace a request.
- Shipping Phase 1 with a small team is realistic.
- API and worker share domain logic with zero duplication and no version skew.
- Module boundaries are real, so extracting a service later is a build-file change plus a
  transport, not a rewrite.

**Bad, accepted**
- API and worker scale together in code size; a worker task loads code it never runs. At
  ~30 MB images this is irrelevant.
- Nothing physically prevents a developer from crossing a module boundary. Mitigated by
  crate-level visibility and CI checks, not by hope.
- A panic in a shared code path can affect both roles — mitigated by separate ECS services
  so a wedged worker cannot consume API capacity.

**Revisit when:** one module's scaling profile genuinely diverges (most likely the AI/worker
path or messaging), or team size passes ~15 engineers and merge contention becomes real.
