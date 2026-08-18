# ADR-0002 — Org-scoped single database with RLS, from day one

**Status:** Accepted · **Date:** 2026-08-18

## Context

Phase 1 has exactly one organization: Lightrees. Phase 3 requires multi-tenancy, and the
proposal's whole "sellable later" argument depends on Phase 3 being cheap.

Options: (a) build single-tenant now, retrofit later; (b) database per tenant;
(c) schema per tenant; (d) shared schema with an `organization_id` discriminator.

## Decision

Option (d), with the tenancy machinery active from the first migration:

- `organization_id NOT NULL` on every business table.
- Composite foreign keys `(organization_id, id)` so a cross-tenant reference violates a
  database constraint.
- PostgreSQL Row-Level Security enabled on all tenant tables, with `SET LOCAL
  app.current_org` per request transaction.
- An application-level `AccessScope` that is the only way to construct a repository query.
- A cross-tenant integration suite running on every PR.

Crucially, all of this is enabled in Phase 1 — with one organization, where it does nothing
visible.

## Consequences

**Good**
- Phase 3 multi-tenancy is an administrative UI plus billing, not a data migration.
- The isolation mechanism is exercised continuously for two phases before a second tenant
  exists. It cannot be "turned on and hope" at the worst possible moment.
- Cross-tenant leakage requires defeating four independent layers.
- One database to back up, monitor, migrate and pay for.

**Bad, accepted**
- Every query carries an `organization_id` predicate and every table an extra column.
  Verbose; caught by lint rather than review.
- Noisy-neighbour effects are possible at scale. Not a pilot concern; the mitigation path
  (read replicas, then per-tenant sharding for the largest tenants) does not require
  changing the model.
- RLS has a small performance cost and can surprise a developer debugging with a raw psql
  session. Documented in the developer guide.
- A single accidental `SET app.current_org` in the wrong scope would be serious. Mitigated
  by setting it only in the request-transaction middleware, with `SET LOCAL` so it cannot
  outlive the transaction.

**Rejected: database per tenant.** Migration fan-out across N databases, N connection
pools, and cross-tenant reporting becomes a data pipeline. Justified for large enterprise
tenants with contractual isolation requirements — revisit if such a customer appears.
