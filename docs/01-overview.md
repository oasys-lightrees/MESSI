# 01 — Overview & Scope

## 1.1 The problem, restated as system requirements

The proposal identifies three operational failures. Each maps to a concrete system
requirement:

| Problem | Symptom | System requirement |
|---|---|---|
| Information scattered | Customer and screening information lives in messages, forms and separate files | A single canonical record per customer/screening, referenced (never copied) by downstream workflows |
| Limited visibility | Managers must ask for updates to learn project health | Project health is *derived* continuously from system state, not reported by humans |
| Manual work | Teams re-summarize, re-create issues, re-enter status | Every transition in the workflow chain pre-fills from the record it came from |

The business impact — slower response, duplicated work, less visibility — is what the KPI
set in [document 10](10-delivery-plan.md) measures.

## 1.2 What MESSI is

An internal operations platform with four surfaces over one shared operational data model:

- **Messenger** — contextual conversations attached to operational objects.
- **Screening** — structured capture that converts conversations and forms into business records.
- **Projects & Issues** — tasks, blockers, ownership, project health.
- **AI Layer** — summarization, classification, extraction, health analysis, assistant, automation triggers.

The AI Layer is deliberately *not* a fifth product surface. It is a capability injected
into the other three, which is why it has no standalone navigation and no standalone KPI.

## 1.3 Users and roles

| Role | Population in pilot | Primary need |
|---|---|---|
| Operator / Coach | 5–15 | Run screenings, work issues, keep records current |
| Project lead | 2–5 | Own delivery, triage blockers, assign work |
| Manager / Exec | 2–3 | Understand health in seconds, without asking |
| Admin | 1–2 | Users, roles, workflow configuration |
| *(Phase 3)* Org admin | — | Tenant-level administration, usage, billing |

Roles are defined formally in [document 08](08-security-and-tenancy.md).

## 1.4 Scope by phase

Mirrors the proposal's MVP staging. Phase boundaries are enforceable: nothing from a later
phase is a prerequisite for an earlier one. Each phase has its own document with scope,
sequencing, exit criteria and risks; the summary below is orientation only.

### [Phase 1 — Internal MVP](11-phase-1-mvp.md)
Screening · Projects · Tasks & issues · Basic dashboard · User and role management.

Delivered as a working internal tool with **no AI in the critical path**. This is
deliberate: it establishes the KPI baseline the proposal asks for, and it proves the
structured-data chain works before any model is involved.

### [Phase 2 — Automation](12-phase-2-automation.md)
Messenger integration · Notifications · AI summaries · AI classification · Project health insights.

AI enters as an *assistive overlay*. Every AI output is attached to a record, attributable,
reviewable and reversible. Turning the AI layer off degrades MESSI to Phase 1 — it never
breaks it.

### [Phase 3 — Productization](13-phase-3-productization.md)
Multi-tenancy · Organization administration · Usage and billing · Custom workflows · External onboarding.

The data model and authorization layer are built Phase-3-ready from the start
([ADR-0002](adr/0002-org-scoped-single-database-multitenancy.md)); Phase 3 is
then mostly UI, billing integration and workflow configuration surfaces — plus the
operating-model commitments that external customers bring.

## 1.5 Explicit non-goals

Recording these protects against the scope creep the proposal names as risk #1.

- **Not a chat replacement.** Messenger is contextual conversation attached to operational
  objects. It does not compete with Slack/Teams for general company chat.
- **Not a CRM.** Customer records exist to support screening and delivery, not pipeline
  management, quotas or marketing automation.
- **Not a BI tool.** The dashboard answers a fixed set of operational questions.
  Ad-hoc analytics is served by exporting to an existing tool.
- **No custom model training.** The AI layer calls hosted models via a provider
  abstraction. Fine-tuning is not in any of the three phases.
- **No autonomous AI actions in Phase 2.** AI proposes; a human or a deterministic rule
  disposes. See [ADR-0005](adr/0005-ai-proposes-humans-dispose.md).
- **No real-time collaborative editing.** Records are edited by one owner at a time with
  optimistic concurrency.

## 1.6 Quality attributes and their targets

Sized for the pilot, with a stated scaling path so the architecture is not accidentally
capped.

| Attribute | Phase 1 target | Scaling path |
|---|---|---|
| Concurrent users | 25 | Horizontal API replicas behind ALB |
| Records | 10⁴ screenings, 10⁵ issues | Partition `events` and `ai_jobs` by month when they exceed 10⁷ rows |
| API latency | p95 < 300 ms for reads, < 600 ms for writes | Read replicas; cache derived health |
| AI latency | Summaries visible < 30 s after trigger; assistant first token < 3 s | Async job model already assumes this |
| Availability | 99.5% business hours | Multi-AZ RDS, ≥2 API tasks |
| RPO / RTO | 24 h / 4 h | PITR backups, IaC-rebuildable environment |
| AI cost | ≤ USD 200 / month at pilot scale | Per-org budget enforcement, caching, model tiering ([doc 06](06-ai-layer.md)) |

## 1.7 Constraints given by the proposal

The technical direction in the proposal is taken as a constraint, not re-litigated:
Next.js frontend · Rust/Actix Web backend · PostgreSQL · Python/FastAPI AI service ·
Docker + AWS. The design below explains *how* these fit together and where the seams are.
Where a choice within those constraints is load-bearing, it is recorded as an ADR.
