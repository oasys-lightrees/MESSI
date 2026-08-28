# 04 — Data Model

PostgreSQL 16. One database, one logical schema (`messi`), plus `pgvector` for embeddings.
Migrations with `sqlx migrate` — plain, ordered, forward-only SQL, reviewed like code.

## 4.1 Conventions

| Convention | Rule |
|---|---|
| Primary keys | `UUID` v7 (time-ordered) — index locality of a sequence, no cross-tenant enumeration |
| Tenancy | Every business table has `organization_id UUID NOT NULL REFERENCES organizations(id)` |
| Timestamps | `TIMESTAMPTZ`, UTC, `created_at` / `updated_at` on all tables |
| Soft delete | `deleted_at TIMESTAMPTZ NULL`; partial indexes carry `WHERE deleted_at IS NULL` |
| Concurrency | `version INTEGER NOT NULL DEFAULT 1`, bumped by trigger on update |
| Enums | Postgres `ENUM` for closed domains that appear in state machines; `TEXT` + `CHECK` for sets expected to grow |
| Money | `NUMERIC(14,4)`; AI cost in micro-USD `BIGINT` to keep accumulation exact |
| Naming | `snake_case`, plural tables, `fk_`/`ix_`/`uq_` prefixes |

Composite foreign keys include `organization_id` so that a cross-tenant reference is
rejected by the database, not merely by application code:

```sql
FOREIGN KEY (organization_id, project_id)
    REFERENCES projects (organization_id, id)
```

This costs a redundant column and buys structural impossibility of the worst bug class in
a multi-tenant system.

## 4.2 Identity and tenancy

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY,
    slug            CITEXT NOT NULL UNIQUE,
    name            TEXT NOT NULL,
    plan            TEXT NOT NULL DEFAULT 'internal',
    settings        JSONB NOT NULL DEFAULT '{}',   -- health thresholds, business calendar, AI toggles
    ai_monthly_budget_micros BIGINT NOT NULL DEFAULT 200000000,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY,
    email           CITEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    password_hash   TEXT,                          -- NULL when SSO-only (Phase 3)
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('invited','active','suspended')),
    mfa_secret      TEXT,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TYPE org_role AS ENUM ('owner','admin','manager','member','viewer');

CREATE TABLE memberships (
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            org_role NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (organization_id, user_id)
);

CREATE TABLE sessions (
    id              UUID PRIMARY KEY,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    refresh_token_hash TEXT NOT NULL,
    user_agent      TEXT,
    ip              INET,
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_sessions_user ON sessions (user_id) WHERE revoked_at IS NULL;
```

`users` is global and `memberships` is the join, so Phase 3's "one person, several client
organizations" needs no restructuring.

## 4.3 Customers

```sql
CREATE TABLE customers (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    kind            TEXT NOT NULL DEFAULT 'company' CHECK (kind IN ('company','individual')),
    external_ref    TEXT,
    attributes      JSONB NOT NULL DEFAULT '{}',
    owner_user_id   UUID REFERENCES users(id),
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,
    UNIQUE (organization_id, id)
);
CREATE UNIQUE INDEX uq_customers_ext ON customers (organization_id, external_ref)
    WHERE external_ref IS NOT NULL AND deleted_at IS NULL;
CREATE INDEX ix_customers_search ON customers
    USING GIN (to_tsvector('simple', name)) WHERE deleted_at IS NULL;

CREATE TABLE contacts (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    customer_id     UUID NOT NULL,
    full_name       TEXT NOT NULL,
    email           CITEXT,
    phone           TEXT,
    role_title      TEXT,
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    FOREIGN KEY (organization_id, customer_id) REFERENCES customers (organization_id, id)
);
```

## 4.4 Screening

> **Superseded by [doc 14](14-followup-engine.md) §14.11.** These tables generalize into
> `followup_modules`, `followup_module_versions`, `followup_cycles` and `followup_answers`;
> the mapping is in [ADR-0007](adr/0007-followup-cycles-generalize-screening.md). The DDL
> below is retained because the column-level decisions — versioned definitions, `origin` and
> `confidence` on every answer, the review-queue indexes — carry over verbatim to the
> generalized tables.


```sql
CREATE TABLE screening_templates (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    key             TEXT NOT NULL,
    version         INTEGER NOT NULL,
    name            TEXT NOT NULL,
    definition      JSONB NOT NULL,   -- sections, questions, types, validation
    scoring_rules   JSONB NOT NULL DEFAULT '{}',
    extraction_schema JSONB,          -- JSON Schema handed to the AI extractor
    field_mapping   JSONB NOT NULL DEFAULT '{}',  -- answer key -> project/customer field
    status          TEXT NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','published','archived')),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organization_id, key, version),
    UNIQUE (organization_id, id)
);

CREATE TYPE screening_status AS ENUM
    ('draft','submitted','in_review','needs_info','accepted','rejected','converted');

CREATE TABLE screenings (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    template_id     UUID NOT NULL,
    customer_id     UUID,
    status          screening_status NOT NULL DEFAULT 'draft',
    score           NUMERIC(8,2),               -- deterministic; never model-written
    score_breakdown JSONB,
    assignee_id     UUID REFERENCES users(id),
    source_type     TEXT CHECK (source_type IN ('form','conversation','import')),
    source_id       UUID,
    attributes      JSONB NOT NULL DEFAULT '{}',
    submitted_at    TIMESTAMPTZ,
    review_started_at TIMESTAMPTZ,
    decided_at      TIMESTAMPTZ,
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,
    FOREIGN KEY (organization_id, template_id)
        REFERENCES screening_templates (organization_id, id),
    FOREIGN KEY (organization_id, customer_id)
        REFERENCES customers (organization_id, id),
    UNIQUE (organization_id, id)
);
CREATE INDEX ix_screenings_queue ON screenings (organization_id, status, submitted_at)
    WHERE deleted_at IS NULL;
CREATE INDEX ix_screenings_assignee ON screenings (organization_id, assignee_id, status)
    WHERE deleted_at IS NULL;

CREATE TABLE screening_answers (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    screening_id    UUID NOT NULL,
    question_key    TEXT NOT NULL,
    value_raw       TEXT,
    value_json      JSONB,            -- normalized typed value
    origin          TEXT NOT NULL DEFAULT 'human'
                    CHECK (origin IN ('human','ai_extracted','imported')),
    confidence      REAL,             -- populated only when origin = 'ai_extracted'
    FOREIGN KEY (organization_id, screening_id)
        REFERENCES screenings (organization_id, id) ON DELETE CASCADE,
    UNIQUE (screening_id, question_key)
);
```

`origin` and `confidence` on every answer mean a reviewer can always see which fields a
model filled in, and the pilot can report "% of fields auto-extracted and accepted
unedited" — the concrete form of the "↓ duplicate data entry" KPI.

## 4.5 Projects, milestones, issues

```sql
CREATE TYPE project_status AS ENUM ('planned','active','on_hold','completed','cancelled');
CREATE TYPE project_health AS ENUM ('unknown','on_track','at_risk','blocked');

CREATE TABLE projects (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    key             TEXT NOT NULL,               -- e.g. 'ALPHA'
    name            TEXT NOT NULL,
    customer_id     UUID,
    status          project_status NOT NULL DEFAULT 'planned',
    health          project_health NOT NULL DEFAULT 'unknown',
    health_computed_at TIMESTAMPTZ,
    progress_pct    SMALLINT NOT NULL DEFAULT 0 CHECK (progress_pct BETWEEN 0 AND 100),
    lead_user_id    UUID REFERENCES users(id),
    starts_on       DATE,
    due_on          DATE,
    source_type     TEXT,                        -- 'screening' | 'manual'
    source_id       UUID,
    context         JSONB NOT NULL DEFAULT '{}', -- unmapped screening answers, carried forward
    attributes      JSONB NOT NULL DEFAULT '{}',
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,
    UNIQUE (organization_id, key),
    UNIQUE (organization_id, id)
);

-- one project per source object; makes conversion idempotent (domain model §3.2)
CREATE UNIQUE INDEX uq_projects_source ON projects (organization_id, source_type, source_id)
    WHERE source_id IS NOT NULL AND deleted_at IS NULL;

CREATE TABLE project_members (
    organization_id UUID NOT NULL,
    project_id      UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    project_role    TEXT NOT NULL CHECK (project_role IN ('lead','contributor','observer')),
    PRIMARY KEY (project_id, user_id),
    FOREIGN KEY (organization_id, project_id) REFERENCES projects (organization_id, id) ON DELETE CASCADE
);

CREATE TABLE milestones (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    project_id      UUID NOT NULL,
    name            TEXT NOT NULL,
    due_on          DATE,
    position        INTEGER NOT NULL DEFAULT 0,
    completed_at    TIMESTAMPTZ,
    FOREIGN KEY (organization_id, project_id) REFERENCES projects (organization_id, id) ON DELETE CASCADE,
    UNIQUE (organization_id, id)
);

CREATE TYPE issue_status AS ENUM ('open','in_progress','blocked','in_review','done','cancelled');
CREATE TYPE issue_severity AS ENUM ('low','medium','high','critical');

CREATE TABLE issues (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    project_id      UUID NOT NULL,
    milestone_id    UUID,
    number          INTEGER NOT NULL,            -- per-project, human-facing: ALPHA-42
    title           TEXT NOT NULL,
    description     TEXT,
    kind            TEXT NOT NULL DEFAULT 'task'
                    CHECK (kind IN ('task','bug','risk','decision','blocker')),
    status          issue_status NOT NULL DEFAULT 'open',
    severity        issue_severity NOT NULL DEFAULT 'medium',
    priority        SMALLINT NOT NULL DEFAULT 2 CHECK (priority BETWEEN 0 AND 3),
    is_blocker      BOOLEAN NOT NULL DEFAULT false,
    assignee_id     UUID REFERENCES users(id),
    reporter_id     UUID REFERENCES users(id),
    estimate_hours  NUMERIC(6,2),
    due_at          TIMESTAMPTZ,
    blocked_since   TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    labels          TEXT[] NOT NULL DEFAULT '{}',
    attributes      JSONB NOT NULL DEFAULT '{}',
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,
    FOREIGN KEY (organization_id, project_id) REFERENCES projects (organization_id, id),
    FOREIGN KEY (organization_id, milestone_id) REFERENCES milestones (organization_id, id),
    UNIQUE (project_id, number),
    UNIQUE (organization_id, id),
    -- domain rule §3.5: in-flight work must be owned
    CONSTRAINT ck_issue_ownership CHECK (
        status NOT IN ('in_progress','blocked','in_review') OR assignee_id IS NOT NULL
    )
);
CREATE INDEX ix_issues_board ON issues (organization_id, project_id, status, priority)
    WHERE deleted_at IS NULL;
CREATE INDEX ix_issues_overdue ON issues (organization_id, due_at)
    WHERE status NOT IN ('done','cancelled') AND deleted_at IS NULL;
CREATE INDEX ix_issues_assignee ON issues (organization_id, assignee_id, status)
    WHERE deleted_at IS NULL;
CREATE INDEX ix_issues_labels ON issues USING GIN (labels);

CREATE TABLE issue_links (
    organization_id UUID NOT NULL,
    from_issue_id   UUID NOT NULL,
    to_issue_id     UUID NOT NULL,
    link_type       TEXT NOT NULL CHECK (link_type IN ('blocks','relates','duplicates','splits_from')),
    PRIMARY KEY (from_issue_id, to_issue_id, link_type),
    CHECK (from_issue_id <> to_issue_id)
);

CREATE TABLE issue_transitions (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    issue_id        UUID NOT NULL,
    from_status     issue_status,
    to_status       issue_status NOT NULL,
    actor_user_id   UUID REFERENCES users(id),
    reason          TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_transitions_issue ON issue_transitions (issue_id, occurred_at);
CREATE INDEX ix_transitions_kpi ON issue_transitions (organization_id, to_status, occurred_at);
```

`issue_transitions` is append-only and is the sole input for cycle-time and
resolution-time KPIs.

## 4.6 Health snapshots

```sql
CREATE TABLE project_health_snapshots (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    project_id      UUID NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    health          project_health NOT NULL,
    progress_pct    SMALLINT NOT NULL,
    open_issues     INTEGER NOT NULL,
    critical_blockers INTEGER NOT NULL,
    overdue_issues  INTEGER NOT NULL,
    days_since_activity INTEGER NOT NULL,
    triggered_rule  TEXT NOT NULL,     -- which rule in §3.4 fired; makes the badge explainable
    metrics         JSONB NOT NULL DEFAULT '{}',
    FOREIGN KEY (organization_id, project_id) REFERENCES projects (organization_id, id) ON DELETE CASCADE
);
CREATE INDEX ix_health_latest ON project_health_snapshots (project_id, computed_at DESC);
```

Snapshots are retained (not overwritten) so the pilot can show health trend over time —
evidence for "↑ management awareness" that a current-state-only design could not produce.

## 4.7 Messaging

```sql
CREATE TABLE conversations (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    subject         TEXT,
    channel         TEXT NOT NULL DEFAULT 'internal'
                    CHECK (channel IN ('internal','email','slack','teams','imported')),
    customer_id     UUID,
    project_id      UUID,
    issue_id        UUID,
    external_ref    TEXT,
    last_message_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ,
    UNIQUE (organization_id, id)
);
CREATE UNIQUE INDEX uq_conversations_external
    ON conversations (organization_id, channel, external_ref)
    WHERE external_ref IS NOT NULL;

CREATE TABLE messages (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    conversation_id UUID NOT NULL,
    author_user_id  UUID REFERENCES users(id),
    author_contact_id UUID,
    author_label    TEXT,               -- for imported messages with no matched identity
    body            TEXT NOT NULL,
    body_format     TEXT NOT NULL DEFAULT 'markdown',
    sent_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    edited_at       TIMESTAMPTZ,
    FOREIGN KEY (organization_id, conversation_id)
        REFERENCES conversations (organization_id, id) ON DELETE CASCADE
);
CREATE INDEX ix_messages_thread ON messages (conversation_id, sent_at);
CREATE INDEX ix_messages_fts ON messages USING GIN (to_tsvector('simple', body));

CREATE TABLE attachments (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entity_type     TEXT NOT NULL,     -- 'message' | 'issue' | 'screening' | 'project'
    entity_id       UUID NOT NULL,
    s3_key          TEXT NOT NULL,
    filename        TEXT NOT NULL,
    content_type    TEXT NOT NULL,
    byte_size       BIGINT NOT NULL,
    checksum_sha256 TEXT NOT NULL,
    scan_status     TEXT NOT NULL DEFAULT 'pending'
                    CHECK (scan_status IN ('pending','clean','infected','error')),
    uploaded_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_attachments_entity ON attachments (organization_id, entity_type, entity_id);
```

## 4.8 AI tables

```sql
CREATE TYPE ai_task_type AS ENUM
    ('summarize','classify','extract','project_health','assistant','embed');
CREATE TYPE ai_job_status AS ENUM
    ('queued','running','succeeded','failed','cancelled','skipped_budget');

CREATE TABLE ai_jobs (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    task_type       ai_task_type NOT NULL,
    subject_type    TEXT NOT NULL,     -- 'screening' | 'project' | 'conversation' | 'issue'
    subject_id      UUID NOT NULL,
    input_digest    TEXT NOT NULL,     -- sha256(normalized input + prompt version); cache key
    prompt_version  TEXT NOT NULL,
    model           TEXT NOT NULL,
    status          ai_job_status NOT NULL DEFAULT 'queued',
    attempts        SMALLINT NOT NULL DEFAULT 0,
    input_tokens    INTEGER,
    output_tokens   INTEGER,
    cost_micros     BIGINT NOT NULL DEFAULT 0,
    latency_ms      INTEGER,
    error           TEXT,
    requested_by    UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at      TIMESTAMPTZ,
    finished_at     TIMESTAMPTZ
);
CREATE INDEX ix_ai_jobs_cache ON ai_jobs (organization_id, task_type, input_digest)
    WHERE status = 'succeeded';
CREATE INDEX ix_ai_jobs_cost ON ai_jobs (organization_id, created_at);
CREATE INDEX ix_ai_jobs_subject ON ai_jobs (subject_type, subject_id, created_at DESC);

CREATE TABLE ai_outputs (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    ai_job_id       UUID NOT NULL REFERENCES ai_jobs(id) ON DELETE CASCADE,
    payload         JSONB NOT NULL,    -- validated against the task's output JSON Schema
    confidence      REAL,
    citations       JSONB NOT NULL DEFAULT '[]',  -- [{entity_type, entity_id, quote}]
    applied_at      TIMESTAMPTZ,       -- NULL until a human or rule promotes it
    applied_by      UUID REFERENCES users(id),
    superseded_by   UUID REFERENCES ai_outputs(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_feedback (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    ai_output_id    UUID NOT NULL REFERENCES ai_outputs(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id),
    disposition     TEXT NOT NULL CHECK (disposition IN ('accepted','edited','rejected')),
    corrected       JSONB,
    note            TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE embeddings (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    chunk_index     INTEGER NOT NULL DEFAULT 0,
    content         TEXT NOT NULL,
    embedding       VECTOR(1024) NOT NULL,
    model           TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (entity_type, entity_id, chunk_index, model)
);
CREATE INDEX ix_embeddings_ann ON embeddings
    USING hnsw (embedding vector_cosine_ops);
CREATE INDEX ix_embeddings_org ON embeddings (organization_id, entity_type);
```

The ANN index is not tenant-partitioned, so retrieval **must** filter by
`organization_id` in the query and post-filter results — enforced by a single repository
function that is the only caller of this table ([doc 08](08-security-and-tenancy.md) §8.6).

## 4.9 Events, outbox and jobs

```sql
CREATE TABLE events (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    event_type      TEXT NOT NULL,      -- 'issue.status_changed', 'screening.submitted', ...
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    actor_user_id   UUID,
    payload         JSONB NOT NULL,     -- before/after diff
    request_id      UUID,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_events_entity ON events (entity_type, entity_id, occurred_at DESC);
CREATE INDEX ix_events_type ON events (organization_id, event_type, occurred_at DESC);

CREATE TABLE outbox (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    event_type      TEXT NOT NULL,
    payload         JSONB NOT NULL,
    available_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    attempts        SMALLINT NOT NULL DEFAULT 0,
    locked_at       TIMESTAMPTZ,
    locked_by       TEXT,
    processed_at    TIMESTAMPTZ,
    last_error      TEXT
);
CREATE INDEX ix_outbox_ready ON outbox (available_at)
    WHERE processed_at IS NULL;

CREATE TABLE jobs (
    id              UUID PRIMARY KEY,
    organization_id UUID,
    kind            TEXT NOT NULL,
    payload         JSONB NOT NULL,
    dedupe_key      TEXT,
    run_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    attempts        SMALLINT NOT NULL DEFAULT 0,
    max_attempts    SMALLINT NOT NULL DEFAULT 5,
    locked_at       TIMESTAMPTZ,
    locked_by       TEXT,
    completed_at    TIMESTAMPTZ,
    failed_at       TIMESTAMPTZ,
    last_error      TEXT
);
CREATE UNIQUE INDEX uq_jobs_dedupe ON jobs (kind, dedupe_key)
    WHERE dedupe_key IS NOT NULL AND completed_at IS NULL;
CREATE INDEX ix_jobs_ready ON jobs (run_at)
    WHERE completed_at IS NULL AND failed_at IS NULL;
```

The claim query, used by both dispatcher and job runner:

```sql
UPDATE jobs SET locked_at = now(), locked_by = $1, attempts = attempts + 1
WHERE id IN (
    SELECT id FROM jobs
    WHERE completed_at IS NULL AND failed_at IS NULL
      AND run_at <= now()
      AND (locked_at IS NULL OR locked_at < now() - INTERVAL '5 minutes')
    ORDER BY run_at
    FOR UPDATE SKIP LOCKED
    LIMIT $2
)
RETURNING *;
```

`SKIP LOCKED` gives safe concurrent consumption across worker replicas with no broker
([ADR-0004](adr/0004-postgres-backed-jobs-and-outbox.md)). The stale-lock interval makes a
crashed worker's jobs recoverable without manual intervention.

## 4.10 Automation and notifications

```sql
CREATE TABLE automation_rules (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    enabled         BOOLEAN NOT NULL DEFAULT true,
    trigger_type    TEXT NOT NULL,     -- event type, or 'schedule'
    trigger_config  JSONB NOT NULL DEFAULT '{}',
    conditions      JSONB NOT NULL DEFAULT '[]',
    actions         JSONB NOT NULL,
    last_fired_at   TIMESTAMPTZ,
    fire_count      BIGINT NOT NULL DEFAULT 0,
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_rules_trigger ON automation_rules (organization_id, trigger_type)
    WHERE enabled;

CREATE TABLE notifications (
    id              UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    kind            TEXT NOT NULL,
    title           TEXT NOT NULL,
    body            TEXT,
    link_url        TEXT,
    entity_type     TEXT,
    entity_id       UUID,
    channels        TEXT[] NOT NULL DEFAULT '{in_app}',
    read_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_notifications_inbox ON notifications (user_id, created_at DESC)
    WHERE read_at IS NULL;
```

## 4.11 Retention and growth

| Table | Growth driver | Policy |
|---|---|---|
| `events` | Every mutation | Retain 24 months; monthly `RANGE` partitioning once > 10⁷ rows |
| `outbox` | Every event | Delete processed rows after 7 days |
| `jobs` | Background work | Delete completed after 30 days; keep failed 90 days |
| `ai_jobs` / `ai_outputs` | AI usage | Retain 24 months — needed for cost reporting and quality evaluation |
| `project_health_snapshots` | 15-min cadence | Keep all for 90 days, then downsample to daily |
| `embeddings` | Re-embedding on edit | Delete on source delete; rebuild on model change |
| `messages` | Conversation volume | Retain per organization policy; default indefinite |

## 4.12 Schema verification

The DDL in this document is not aspirational — it was applied to a live PostgreSQL 16
instance and the guarantees it claims were exercised:

| Check | Result |
|---|---|
| Full schema applies from empty database | 28 tables, 0 errors |
| Cross-tenant reference (issue in org B → project in org A) | Rejected by `issues_organization_id_project_id_fkey` |
| `in_progress` issue with no assignee | Rejected by `ck_issue_ownership` |
| Same issue with an assignee | Accepted |
| Second project converted from one screening | Rejected by `uq_projects_source` |
| RLS: app role scoped to org A / org B | Sees only its own rows |
| RLS: no `app.current_org` set | Errors — fails closed (see [doc 08](08-security-and-tenancy.md) §8.4) |
| Duplicate pending job with same `dedupe_key` | Rejected by `uq_jobs_dedupe` |
| Two concurrent workers running the claim query | 3 rows each, zero double-claims |

`pgvector` was not available in the verification environment, so the `embeddings` vector
column and its HNSW index are the one part of the schema validated by inspection only.

## 4.13 Migration and seeding practice

- Forward-only migrations; a mistake is corrected by a new migration, never by editing a
  shipped one.
- Expand/contract for breaking changes: add column → backfill in a job → switch reads →
  drop old column in a later release. No deployment requires downtime.
- `CREATE INDEX CONCURRENTLY` for indexes added to populated tables.
- `sqlx::query!` compile-time verification against a checked-in `.sqlx` offline snapshot,
  so schema drift breaks the build rather than production.
- Seed data ships per environment: one organization, the role matrix, two screening
  templates and a demo project so a reviewer can see the whole chain in a fresh checkout.
