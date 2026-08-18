# 10 — Delivery Plan & KPIs

## 10.1 Principle

> "Success should be measured by operational improvement, not by the amount of AI used."
> — proposal, slide 7

Every KPI below has a named table or metric behind it, and the **baseline is captured
before the feature that is meant to improve it ships**. A KPI whose measurement starts
after the intervention cannot demonstrate anything.

## 10.2 Phase 1 — Internal MVP

**Goal:** the screening → project → issue chain works, is adopted, and produces the KPI
baseline.

| Workstream | Deliverable |
|---|---|
| Foundation | Repo, CI, Terraform for staging, Postgres schema, migrations, seed data |
| Identity | Login, MFA, sessions, roles, memberships, invitations, audit log |
| Customers | Customer + contact CRUD, timeline view |
| Screening | Template builder (versioned), form runtime, submission, review queue, deterministic scoring, accept/reject |
| Conversion | Screening → project, with declarative field mapping and provenance |
| Projects | CRUD, membership, milestones, lifecycle transitions |
| Issues | CRUD, board, transitions with ownership enforcement, comments, links, transition audit |
| Health (deterministic) | Snapshot job, rule evaluation, health badge, progress % |
| Dashboard | Portfolio view, my-work view, screening queue |
| Platform | Events, outbox, job runner, notifications (in-app + email) |
| Instrumentation | KPI aggregation job and `GET /v1/insights/kpis` |

**Explicitly not in Phase 1:** any model call. The AI service is scaffolded and deployed
with health checks, and it is not on any user path.

### Exit criteria

| # | Criterion |
|---|---|
| 1 | Two real Lightrees workflows run end-to-end in production for four consecutive weeks |
| 2 | ≥ 80% of pilot users active weekly |
| 3 | Zero known cross-tenant or authorization defects; isolation suite green |
| 4 | Baselines captured for all six KPIs below, with ≥ 30 completed screenings and ≥ 100 closed issues |
| 5 | p95 API latency within the [doc 01](01-overview.md) §1.6 targets |
| 6 | Users report the chain removes re-entry (structured post-pilot interview, not a vibe) |

Criterion 4 is the gate that matters. Phase 2 cannot be evaluated without it.

## 10.3 Phase 2 — Automation

**Goal:** AI measurably improves the Phase 1 baseline, at controlled cost.

| Workstream | Deliverable |
|---|---|
| Messenger | Conversations attached to objects, message posting, SSE streaming, email ingestion |
| Promotion | Conversation → screening with AI extraction pre-fill |
| AI: extract | Field extraction with per-field confidence, review UI, `apply-extraction` |
| AI: classify | Screening and issue classification as proposal; confidence-gated auto-apply rule |
| AI: summarize | Conversation and project-update summaries with citations |
| AI: project health | Narrative + ranked recommended actions on the health card |
| AI: assistant | Retrieval-grounded Q&A over authorized data, streamed with citations |
| Automation | Rules engine, condition/action catalogue, dry-run testing, escalation rules |
| Notifications | Preferences, digests, quiet hours, Slack/Teams adapter |
| AI operations | Budgets, caching, usage dashboard, feedback capture, eval suite in CI |

### Rollout discipline

Each AI feature ships behind a flag and follows the same four steps:

1. **Shadow** — job runs, output stored, nothing shown. Compare against human outcomes.
2. **Suggest** — shown as a proposal with confidence and citations. Capture `ai_feedback`.
3. **Assist** — pre-filled and one-click acceptable, where edit-free acceptance ≥ 70%.
4. **Auto** *(classification only, confidence-gated)* — auto-applied above the threshold,
   still recorded and reversible.

Nothing skips a step. A feature that stalls at step 2 with a poor acceptance rate has
produced a useful finding — that this task is not worth AI — which is exactly what the
pilot is for.

### Exit criteria

| # | Criterion |
|---|---|
| 1 | ≥ 3 KPIs improved against baseline by the target margins below |
| 2 | Edit-free acceptance ≥ 70% for extraction and classification |
| 3 | AI cost within budget, cost-per-screening documented |
| 4 | Zero incidents of AI output causing an incorrect business outcome |
| 5 | Managers report the health card answers their question without follow-up |
| 6 | Eval suite green, with each production prompt version's scores recorded |

## 10.4 Phase 3 — Productization

Entered only if Phase 2 exit criteria are met. Multi-tenancy (mostly already latent),
organization administration, usage metering and billing, configurable workflows,
self-service onboarding, SSO/SCIM, public API and webhooks.

The design decisions that make this phase cheap were made in Phase 1: `organization_id`
everywhere with RLS ([ADR-0002](adr/0002-org-scoped-single-database-multitenancy.md)),
data-driven automation rules ([doc 07](07-events-and-automation.md) §7.5), declarative
screening templates and field mappings, and per-org AI budgets already metered.

## 10.5 The KPI scorecard

Six KPIs, mapped to the proposal's three value themes. Each has a definition precise
enough that two people compute the same number.

### Reduce manual work

| KPI | Definition | Source | Target |
|---|---|---|---|
| **Screening processing time** | Median hours from `screenings.submitted_at` to `decided_at`, business hours only | `screenings` | −40% by end of Phase 2 |
| **Duplicate data entry** | Share of project/customer fields populated by conversion or accepted extraction rather than typed | `screening_answers.origin`, `projects.source_id`, `ai_feedback` | ≥ 60% auto-populated |

### Improve visibility

| KPI | Definition | Source | Target |
|---|---|---|---|
| **Project status reporting time** | Median minutes spent producing the weekly status report (timed exercise, 5 projects, same reporter, monthly) | Manual measurement protocol | −70% |
| **Issue ownership** | Share of issues in `in_progress`/`blocked`/`in_review` with an assignee | `issues` | 100% (structurally enforced, so this measures rule integrity) |

### Improve response

| KPI | Definition | Source | Target |
|---|---|---|---|
| **Issue resolution time** | Median business hours from first `open` transition to `done` | `issue_transitions` | −25% |
| **Overdue issues** | Count of open issues past `due_at`, sampled daily | `issues` daily rollup | −50% |

### Adoption and cost, tracked alongside

| Metric | Definition |
|---|---|
| Weekly active users | Distinct users with ≥ 1 state-changing event in the week |
| Records created per active user per week | Screenings + issues + comments |
| Edit-free AI acceptance rate | `ai_feedback.disposition='accepted'` ÷ all dispositions, by task type |
| AI cost per screening / per project per month | `ai_jobs.cost_micros` ÷ volume |

### Measurement mechanics

- `kpi.rollup` runs nightly, materializing per-day values into `kpi_daily` so a
  dashboard query is a range scan, not a recomputation over `issue_transitions`.
- Business-hours arithmetic uses the organization's calendar from
  `organizations.settings` — a blocker opened Friday 17:00 and resolved Monday 09:00 is
  two business hours, not sixty-four. Reporting elapsed clock time would make every
  weekend look like a crisis.
- KPI definitions are versioned. If a definition changes, the chart shows the change point
  rather than silently rewriting history.
- Two KPIs (reporting time, and the qualitative exit criteria) are **manual measurements
  by protocol**. Pretending everything is automatable produces either a bad proxy metric or
  a missing one; a documented monthly 20-minute exercise is more honest and more useful.

## 10.6 Risk register

The proposal's three risks, plus what this design actually does about them.

| Risk | Control in this design |
|---|---|
| **Scope creep** | Non-goals written down ([doc 01](01-overview.md) §1.5); phase exit criteria are gates, not suggestions; nothing from a later phase is a prerequisite for an earlier one |
| **AI cost** | Budget ceiling enforced pre-dispatch; content-addressed caching; model tiering; debouncing; per-task cost visible on a dashboard from the first AI feature |
| **Adoption** | Phase 1 delivers value with no AI, so adoption is not contingent on model quality; conversion flows remove re-entry immediately; four-step AI rollout means users never meet an unreliable feature presented as reliable |
| *(added)* **AI quality is unmeasurable** | `ai_feedback` captured at the point of use; golden-set evals gate every prompt change; edit-free acceptance is a named KPI |
| *(added)* **Health metrics become distrusted** | Deterministic metrics separated from AI narrative; `triggered_rule` recorded so every badge is explainable; thresholds configurable per organization |
| *(added)* **Automation misfires** | Dry-run against 30 days of history, per-rule fire budget, cascade-hop limit, automation-origin marking |
| *(added)* **Key-person dependency** | One repository, `make dev` under 10 minutes, ADRs recording why, runbooks written before incidents |

## 10.7 First decisions needed

To start Phase 1, three inputs are required from the business — none of them technical:

1. **Which two Lightrees workflows** are the pilot? The screening templates and the field
   mappings are built for these specifically.
2. **Who are the 10–15 pilot users**, and which of them are managers? Role assignment and
   the dashboard's default view depend on it.
3. **What are today's numbers** for the six KPIs, even roughly? If a pre-system baseline
   cannot be reconstructed, the first four weeks of Phase 1 become the baseline period —
   which is acceptable, but it must be a decision rather than a discovery.

The proposal's recommended next step — "approve a small internal MVP focused on Screening +
Projects + Issues" — is exactly Phase 1 above.
