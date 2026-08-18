# 05 — API Design

REST over HTTPS, JSON bodies, versioned at `/v1`. OpenAPI 3.1 is generated from the Rust
handlers (`utoipa`) and is the contract from which the frontend's TypeScript client is
generated. Drift between code and spec fails CI.

## 5.1 Conventions

| Concern | Decision |
|---|---|
| Base path | `/v1`; breaking changes create `/v2`, additive changes do not bump |
| Auth | `Authorization: Bearer <access_token>`; short-lived (15 min), refreshed via rotating refresh token |
| Tenancy | Derived from the token. **Never** accepted from a path, query or header. |
| IDs | UUID v7 strings |
| Time | RFC 3339, UTC, always with offset |
| Casing | `snake_case` in JSON, matching the database and the Rust DTOs |
| Concurrency | `If-Match: "<version>"` on updates → `409 Conflict` on mismatch |
| Idempotency | `Idempotency-Key` header honored on all `POST`s that create or trigger work |
| Partial update | `PATCH` with merge semantics; explicit `null` clears, absent leaves unchanged |
| Rate limits | Per user and per organization; `429` with `Retry-After` and `RateLimit-*` headers |

## 5.2 Errors

One envelope for every failure — RFC 9457 Problem Details, extended with a stable machine
code and per-field detail:

```json
{
  "type": "https://messi.lightrees.internal/errors/validation_failed",
  "title": "Validation failed",
  "status": 422,
  "code": "validation_failed",
  "detail": "2 fields are invalid.",
  "request_id": "018f3c2a-...",
  "errors": [
    { "field": "due_at",      "code": "in_the_past", "message": "Due date must be in the future." },
    { "field": "assignee_id", "code": "not_a_member", "message": "User is not a member of this project." }
  ]
}
```

| Status | Used for |
|---|---|
| 400 | Malformed request |
| 401 | Missing/expired credentials |
| 403 | Authenticated but not permitted |
| 404 | Not found **or** not visible to the caller (never distinguish — that leaks existence) |
| 409 | Version conflict, or illegal state transition |
| 422 | Validation failure with field detail |
| 429 | Rate or budget limit |
| 503 | Downstream dependency (AI provider) unavailable; `Retry-After` set |

An illegal state transition returns `409` with `code: "invalid_transition"` and lists the
transitions that *are* legal from the current state — the client renders correct buttons
without duplicating the state machine.

## 5.3 Collections

```
GET /v1/issues?project_id=…&status=open,in_progress&assignee_id=me
              &due_before=2026-09-01&q=credentials
              &sort=-priority,due_at&limit=50&cursor=eyJ…
```

- **Cursor pagination** (keyset) everywhere. Offset pagination is not offered; it is
  inconsistent under concurrent writes and degrades on deep pages.
- Response envelope:

```json
{
  "data": [ /* … */ ],
  "page": { "next_cursor": "eyJ…", "has_more": true },
  "meta": { "total_estimate": 128 }
}
```

`total_estimate` is explicitly an estimate (from `reltuples`/`count` with a cap); exact
counts on large filtered sets are a per-request table scan for a number nobody acts on.

- `?fields=` for sparse responses; `?expand=assignee,project` for a bounded set of
  inlineable relations. Expansion depth is 1 — deeper needs are a purpose-built endpoint,
  not a query language.

## 5.4 Endpoint surface

### Identity
```
POST   /v1/auth/login                     email + password (+ MFA)
POST   /v1/auth/refresh                   rotating refresh token
POST   /v1/auth/logout
GET    /v1/me                             user, org, role, effective permissions
GET    /v1/users                          admin
POST   /v1/users/invite                   admin
PATCH  /v1/users/{id}                     admin — role, status
```

### Customers
```
GET    /v1/customers
POST   /v1/customers
GET    /v1/customers/{id}
PATCH  /v1/customers/{id}
GET    /v1/customers/{id}/timeline        merged screenings, projects, conversations
```

### Screening
```
GET    /v1/screening-templates
POST   /v1/screening-templates
POST   /v1/screening-templates/{id}/publish
POST   /v1/screening-templates/{id}/versions

GET    /v1/screenings                     ?status=&assignee_id=&template_key=
POST   /v1/screenings
GET    /v1/screenings/{id}
PATCH  /v1/screenings/{id}
PUT    /v1/screenings/{id}/answers        bulk upsert, validated against the template version
POST   /v1/screenings/{id}/submit
POST   /v1/screenings/{id}/transition     { to: accepted|rejected|needs_info, reason }
POST   /v1/screenings/{id}/convert        → project   (idempotent per screening)

GET    /v1/screenings/{id}/ai/extraction  proposed field values, per-field confidence
POST   /v1/screenings/{id}/ai/extraction  request (re-)extraction
POST   /v1/screenings/{id}/apply-extraction
       { accept: ["budget","timeline"], edits: { "sector": "Education" }, reject: ["headcount"] }
```

`apply-extraction` is the human-in-the-loop gate. It writes the accepted values into
`screening_answers` with `origin='ai_extracted'`, and writes one `ai_feedback` row per
field. One endpoint call therefore updates the business record *and* the quality dataset —
they cannot fall out of sync.

### Projects
```
GET    /v1/projects                       ?status=&health=&lead_user_id=&customer_id=
POST   /v1/projects
GET    /v1/projects/{id}
PATCH  /v1/projects/{id}
POST   /v1/projects/{id}/transition       { to: active|on_hold|completed|cancelled, reason }
GET    /v1/projects/{id}/health           latest snapshot + rule that fired + AI narrative
GET    /v1/projects/{id}/health/history   ?since=
POST   /v1/projects/{id}/health/recompute manual refresh (rate-limited)
GET    /v1/projects/{id}/members
PUT    /v1/projects/{id}/members/{user_id}
DELETE /v1/projects/{id}/members/{user_id}
GET    /v1/projects/{id}/milestones
POST   /v1/projects/{id}/milestones
```

`GET /v1/projects/{id}/health` is the slide-6 payload:

```json
{
  "project": { "id": "…", "key": "ALPHA", "name": "AlphaLeaders MVP" },
  "computed_at": "2026-08-18T09:15:00Z",
  "health": "at_risk",
  "progress_pct": 68,
  "metrics": {
    "open_issues": 4,
    "critical_blockers": 2,
    "overdue_issues": 1,
    "days_since_activity": 1
  },
  "triggered_rule": "milestone_at_risk",
  "risks": [
    { "issue_id": "…", "title": "Client approval delayed",        "severity": "critical" },
    { "issue_id": "…", "title": "Missing integration credentials", "severity": "critical" }
  ],
  "ai_summary": {
    "text": "The project is currently at risk because approval is delayed and the integration is blocked by missing credentials.",
    "recommended_actions": [
      { "text": "Escalate approval",        "issue_id": "…" },
      { "text": "Assign integration owner", "issue_id": "…" },
      { "text": "Review UAT impact",        "issue_id": null }
    ],
    "generated_at": "2026-08-18T09:15:04Z",
    "model": "claude-sonnet-…",
    "stale": false
  }
}
```

Note the shape: every deterministic number sits outside `ai_summary`, and `ai_summary` is
nullable. A client renders a complete, correct card when the AI layer is down.

### Issues
```
GET    /v1/issues
POST   /v1/issues
GET    /v1/issues/{id}
PATCH  /v1/issues/{id}
POST   /v1/issues/{id}/transition         { to, reason?, blocking_issue_id? }
POST   /v1/issues/{id}/assign             { user_id }
GET    /v1/issues/{id}/comments
POST   /v1/issues/{id}/comments
POST   /v1/issues/{id}/links              { to_issue_id, link_type }
POST   /v1/issues/bulk                    bounded batch transition/assign/label
```

### Messaging
```
GET    /v1/conversations                  ?project_id=&customer_id=&issue_id=
POST   /v1/conversations
GET    /v1/conversations/{id}/messages    cursor-paginated, ascending
POST   /v1/conversations/{id}/messages
POST   /v1/conversations/{id}/promote     → screening (idempotent per conversation)
GET    /v1/conversations/{id}/stream      SSE: new messages, typing, AI status
```

### AI
```
POST   /v1/ai/tasks                       { task_type, subject_type, subject_id, options }
                                          → 202 { job_id, cached: bool }
GET    /v1/ai/jobs/{id}
GET    /v1/ai/outputs?subject_type=&subject_id=
POST   /v1/ai/outputs/{id}/feedback       { disposition, corrected?, note? }
POST   /v1/ai/assistant/query             SSE stream: tokens, then citations, then usage
GET    /v1/ai/usage?from=&to=&group_by=task_type
```

### Automation, notifications, insights
```
GET    /v1/automation/rules
POST   /v1/automation/rules
POST   /v1/automation/rules/{id}/test     dry run against recent events; returns would-fire list
PATCH  /v1/automation/rules/{id}
GET    /v1/notifications                  ?unread=true
POST   /v1/notifications/{id}/read
POST   /v1/notifications/read-all

GET    /v1/insights/dashboard             manager landing payload, single round trip
GET    /v1/insights/kpis?from=&to=        the pilot scorecard (doc 10)
POST   /v1/exports                        { kind, filters, format } → 202 job → signed S3 URL
```

`POST /v1/automation/rules/{id}/test` exists because an automation rule that silently
misfires across a live workflow is the fastest way to lose user trust in the platform.

## 5.5 Streaming

Server-Sent Events, not WebSockets: the traffic is server→client, SSE survives proxies
without special configuration, and it reconnects natively with `Last-Event-ID`.

```
event: token
data: {"text":"The project is "}

event: citation
data: {"entity_type":"issue","entity_id":"…","quote":"approval pending since 2026-08-04"}

event: done
data: {"job_id":"…","input_tokens":2210,"output_tokens":180,"cost_micros":8400}
```

## 5.6 Internal AI service contract

Not exposed publicly; reachable only from the private subnet, authenticated with a signed
service token, and additionally pinned by security group.

```
POST /internal/tasks/summarize
POST /internal/tasks/classify
POST /internal/tasks/extract
POST /internal/tasks/project-health
POST /internal/tasks/assistant      (streaming)
POST /internal/embeddings
GET  /internal/health               liveness + provider reachability
GET  /internal/prompts              registry: task → prompt version → schema hash
```

Request and response envelopes are fixed across task types:

```json
// request
{
  "task_type": "extract",
  "prompt_version": "extract.v3",
  "input": { "text": "…", "target_schema": { /* JSON Schema */ } },
  "context": [ { "kind": "message", "id": "…", "text": "…" } ],
  "constraints": { "max_output_tokens": 800, "model_tier": "standard", "timeout_ms": 20000 }
}

// response
{
  "output": { /* conforms to target_schema */ },
  "confidence": 0.82,
  "citations": [ { "kind": "message", "id": "…", "quote": "…" } ],
  "usage": { "input_tokens": 1840, "output_tokens": 260, "model": "…", "cost_micros": 7100 },
  "prompt_version": "extract.v3"
}
```

The AI service is **stateless and idempotent**: same request → same cost accounting path,
no writes, no side effects. Retries are therefore always safe, which is what lets the
worker's retry policy be simple.

## 5.7 Webhooks (Phase 3)

Outbound only, HMAC-SHA256 signed with a per-endpoint secret, `t=` timestamp in the
signature header to prevent replay, at-least-once delivery from the same outbox, and
exponential backoff to 24 h before an endpoint is auto-disabled with an admin notification.
