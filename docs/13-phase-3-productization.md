# 13 — Phase 3: Productization

> Proposal, slide 8: *Multi-tenancy · Organization administration · Usage / billing · Custom workflows · External customer onboarding*
> Proposal, slide 10: *Build internal value first, then reuse the platform for other organizations.*

**Goal:** turn a validated internal tool into a platform a second organization can use
without Lightrees-specific assumptions baked in.

**Entry condition:** all six [Phase 2](12-phase-2-automation.md) exit criteria met. This
phase is entered only on evidence, because it is the phase that stops being cheap —
external customers bring support obligations, contractual commitments and a compatibility
surface that cannot be walked back.

## 13.1 Scope

| Workstream | Deliverable |
|---|---|
| Multi-tenancy | Organization switching, cross-org user membership, tenant-scoped admin |
| Organization administration | Settings UI, role management, health-rule thresholds, AI toggles and budgets per tenant |
| Usage & billing | Metering, plan limits, invoicing integration, usage dashboards |
| Custom workflows | Screening template designer, configurable field mappings, per-tenant automation rule library |
| Onboarding | Self-service signup, guided setup, template gallery, sample data |
| Identity | OIDC/SAML SSO, SCIM provisioning |
| Integration | Public API, outbound webhooks, API keys and scopes |

## 13.2 Why this phase is mostly UI

The expensive parts of multi-tenancy were done in Phase 1, when they were free:

| Already in place | Since | What Phase 3 adds |
|---|---|---|
| `organization_id` on every table, composite FKs | Phase 1 migration 1 | Nothing — already correct |
| Row-level security, exercised daily | Phase 1 | Nothing — already correct |
| `users` global with `memberships` join | Phase 1 | Org switcher UI |
| Per-org AI budgets, metered `cost_micros` | Phase 2 | Billing integration on top of existing meters |
| Health thresholds in `organizations.settings` | Phase 1 | Admin UI to edit them |
| Automation rules as data | Phase 2 | Per-tenant rule library, template gallery |
| Versioned screening templates with declarative field mapping | Phase 1 | Designer UI |

This is the payoff for [ADR-0002](adr/0002-org-scoped-single-database-multitenancy.md) and
for keeping automation and templates data-driven. The remaining work is genuinely new
product surface — billing, onboarding, SSO, public API — rather than retrofitting.

**The honest caveat:** "mostly UI" is a claim about the data model, not about total effort.
Billing, SSO/SCIM and a public API each carry their own correctness and support burden, and
together they are likely comparable in size to Phase 2.

## 13.3 What changes in the operating model

The technical work is the smaller half of this phase. External customers change obligations
that Phases 1 and 2 did not have:

| Area | Internal (Phases 1–2) | External (Phase 3) |
|---|---|---|
| Availability | 99.5% business hours, best effort | Contractual SLA, out-of-hours response |
| Support | Ask the team | Ticketing, response targets, escalation path |
| Data | One organization's own data | Processor obligations, DPAs, sub-processor disclosure |
| Breaking changes | Coordinate internally | Versioned API, deprecation policy, migration notice |
| Incidents | Internal comms | Status page, customer notification duties |
| Security | Internal review | Customer security questionnaires, likely a penetration test |

These are decisions for the business, not the engineering team, and they should be settled
before the first external customer rather than discovered by them.

## 13.4 Standardization: separating Lightrees from the platform

Slide 10's step 2 — "separate company-specific workflows from reusable platform features" —
is a concrete audit, run at the start of this phase:

1. Inventory every hard-coded assumption: screening question keys, health thresholds,
   role names, notification templates, issue types, business calendar.
2. For each, decide: platform default, tenant configuration, or Lightrees-only.
3. Move tenant configuration into `organizations.settings` or the template/rule tables.
4. Delete anything Lightrees-only that no longer has a user.

The audit is cheap because most of these are already configuration. Its value is in finding
the ones that are not — and those are best found deliberately, rather than by a second
tenant hitting them.

## 13.5 Not in this phase

| Excluded | Why |
|---|---|
| Database-per-tenant isolation | Shared schema with RLS is sufficient until a customer contractually requires physical isolation ([ADR-0002](adr/0002-org-scoped-single-database-multitenancy.md)) |
| Custom model training per tenant | Not in any phase |
| White-labelling / custom domains | Not required to validate the commercial hypothesis |
| Marketplace or third-party plugins | Far beyond the evidence available at this point |

## 13.6 Exit criteria

| # | Criterion |
|---|---|
| 1 | A second organization is onboarded self-service, with no Lightrees engineer in the loop |
| 2 | Cross-tenant isolation suite green, plus an independent penetration test with no high findings |
| 3 | Usage metering reconciles with provider invoices within 2% |
| 4 | A tenant configures a screening template and an automation rule without support |
| 5 | Public API documented, versioned, with a published deprecation policy |
| 6 | Support, SLA and incident-response processes defined and rehearsed |

Criterion 1 is the real test of slide 10. If onboarding a second organization needs an
engineer, the platform is still a bespoke internal tool with a login screen.

## 13.7 Phase-specific risks

| Risk | Signal | Response |
|---|---|---|
| Configurability without limit | Every prospect requires a new setting | Configuration surface is a product decision with an owner; the audit in §13.4 produces a fixed list, and additions need justification |
| First external customer drives the roadmap | Backlog dominated by one tenant's needs | Distinguish "platform gap" from "this customer's workflow"; the latter is consulting revenue, not product scope |
| Noisy neighbour on shared infrastructure | Latency variance across tenants | Per-tenant rate limits and AI budgets already exist; read replicas next; per-tenant sharding is the documented path if it comes to that |
| Internal Lightrees needs get deprioritised | Phase 1–2 users report regressions | Lightrees is a tenant with the same escalation path as any other, not an afterthought |
| Support load underestimated | Response times slipping | §13.3 is settled before the first external customer, not after |

## 13.8 What this phase needs to start

- Phase 2 exit criteria signed off, with the KPI improvements and AI cost figures published
  — this is the evidence the commercial hypothesis rests on.
- A commercial decision on pricing model and target segment. The proposal names coaching,
  consulting, education, HR and training as candidates; metering design depends on which.
- Business sign-off on the operating-model commitments in §13.3.
