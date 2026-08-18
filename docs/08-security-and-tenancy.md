# 08 — Security & Tenancy

## 8.1 Threat model

| Actor | Concern | Primary control |
|---|---|---|
| Internal user | Sees data outside their remit (other projects, HR-sensitive screenings) | RBAC + per-project scoping, enforced in the query layer |
| Internal user | Escalates own role | Role changes are admin-only, audited, and cannot be self-applied |
| Compromised credential | Full account takeover | MFA, short access tokens, refresh rotation with reuse detection, session revocation |
| Phase 3 tenant | Reads another tenant's data | `organization_id` on every row, composite FKs, RLS, per-request scoped connection |
| Customer-supplied content | Prompt injection reaching data or actions | AI service holds no credentials and no tools ([doc 06](06-ai-layer.md) §8) |
| Attachment upload | Malware distribution, stored XSS | Type allow-list, AV scan before download availability, `Content-Disposition: attachment`, served from a separate origin |
| Provider | Operational data retained/trained on | Zero-retention configuration, redaction, per-org disable switch |

## 8.2 Authentication

**Phase 1**
- Email + password. Argon2id (m=64 MiB, t=3, p=1), per-user salt.
- TOTP MFA, mandatory for `owner`, `admin` and `manager`.
- Access token: JWT, 15 min, `EdDSA` (Ed25519), claims `sub`, `org`, `role`, `sid`, `exp`, `jti`.
- Refresh token: opaque, 30 days, stored hashed, **rotated on every use**. Reuse of a
  consumed refresh token revokes the entire session family and alerts — the standard
  detection for a stolen refresh token.
- Session cookie is httpOnly, `Secure`, `SameSite=Lax`, held by the Next.js server. The
  browser never holds an API token.
- Login throttling: exponential per-account delay, IP-based rate limiting, generic failure
  message regardless of whether the account exists.

**Phase 3** adds OIDC/SAML SSO with SCIM provisioning; `users.password_hash` becomes `NULL`
for federated accounts and the local path is disabled per organization.

## 8.3 Authorization

Two dimensions, both required.

**Organization role** — coarse capability:

| Capability | owner | admin | manager | member | viewer |
|---|:---:|:---:|:---:|:---:|:---:|
| Manage organization settings | ✔ | ✔ | | | |
| Manage users and roles | ✔ | ✔ | | | |
| Manage screening templates | ✔ | ✔ | ✔ | | |
| Create/convert screenings | ✔ | ✔ | ✔ | ✔ | |
| Create projects | ✔ | ✔ | ✔ | | |
| Edit any project | ✔ | ✔ | ✔ | | |
| Edit projects they are a member of | ✔ | ✔ | ✔ | ✔ | |
| Transition issues | ✔ | ✔ | ✔ | ✔ | |
| View portfolio dashboard | ✔ | ✔ | ✔ | ✔ | ✔ |
| Manage automation rules | ✔ | ✔ | ✔ | | |
| View AI usage and cost | ✔ | ✔ | ✔ | | |
| Export data | ✔ | ✔ | ✔ | | |

**Project membership** — object scope. A `member` sees projects they belong to, plus
projects marked `visibility='org'`. `manager` and above see all projects in the
organization. Screenings are visible to their assignee, to the creator and to
`manager`+, because screening content is frequently the most sensitive data in the system.

### Enforcement

Authorization is enforced in the **repository/query layer**, not in handlers:

```rust
// Every list query goes through a scope, which is constructed only from the
// authenticated principal. There is no repository method that takes a raw org id.
let scope = AccessScope::for_principal(&principal);   // org + visible project ids + role
let issues = self.issues.list(&scope, filters).await?;
```

Handler-level checks are additive defence, never the sole gate — a forgotten `if` in one
of eighty handlers is the realistic failure mode, so the design removes the possibility of
writing an unscoped query rather than relying on reviewers to spot one.

A CI lint rejects any `sqlx` query against a tenant-scoped table that does not bind an
`organization_id` parameter.

## 8.4 Tenancy isolation

Defence in depth, four layers:

1. **Schema** — `organization_id NOT NULL` on every business table; composite foreign keys
   that make a cross-tenant reference a constraint violation ([doc 04](04-data-model.md) §4.1).
2. **Application** — `AccessScope` as above.
3. **Database** — PostgreSQL Row-Level Security. The application connects as a non-owner
   role; each request sets `SET LOCAL app.current_org = '<uuid>'` inside its transaction:

```sql
ALTER TABLE issues ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON issues
    USING (organization_id = current_setting('app.current_org')::uuid);
```

   Two details that matter and are easy to get wrong:

   - **The application role must not own the tables.** A table owner bypasses its own RLS
     policies. Migrations run as a `messi_migrator` role that owns the schema; the runtime
     connects as `messi_app`, which owns nothing. `FORCE ROW LEVEL SECURITY` is set as a
     second line of defence.
   - **A missing `app.current_org` fails closed and loudly.** `current_setting` without the
     `missing_ok` flag yields an empty string, and the cast to `uuid` raises. A request that
     somehow reaches the database without passing through the tenancy middleware therefore
     errors rather than returning a silently unfiltered — or silently empty — result set.
     Using `current_setting('app.current_org', true)` instead would return `NULL`, match no
     rows, and turn a serious bug into an invisible one. We deliberately do not do that.

   RLS is enabled from Phase 1 even with a single organization. Turning it on later, against
   a populated database and eighty existing queries, is a migration nobody wants; turning it
   on now costs nothing and means the Phase 3 switch is already tested every day.

4. **Test** — a cross-tenant integration suite seeds two organizations and asserts that
   every list endpoint, every get-by-id and every retrieval query returns nothing for the
   other tenant. This suite runs on every PR.

## 8.5 Data protection

| Layer | Control |
|---|---|
| In transit | TLS 1.3 externally; TLS internally between all services; HSTS with preload |
| At rest | RDS encryption (KMS CMK), S3 SSE-KMS, encrypted EBS |
| Field level | `users.mfa_secret`, provider API keys and webhook secrets encrypted with an application key from AWS Secrets Manager — an RDS snapshot leak does not yield working credentials |
| Backups | Automated daily + PITR (7 days), encrypted, restore tested quarterly |
| Secrets | AWS Secrets Manager, rotated; never in environment files in the repository; CI has no production secrets |
| Logs | Structured, PII-scrubbed by a field allow-list; message bodies and screening answers are **never** logged |
| Attachments | Signed, short-lived S3 URLs; AV scan gate before the URL is issuable |

### PII and GDPR

- **Inventory.** Tables holding personal data are annotated in the migration files and
  listed in a generated register.
- **Erasure.** `DELETE /v1/admin/customers/{id}/erase` hard-deletes personal fields,
  tombstones the record, and deletes derived artefacts: embeddings, AI outputs and
  cached AI results referencing it. Erasure that leaves the data sitting in an embedding
  table is not erasure.
- **Export.** Per-subject export as JSON via the same export pipeline.
- **Retention.** Enforced by the daily job in [doc 07](07-events-and-automation.md) §7.4.
- **Processors.** The model provider is a sub-processor and must be listed; zero-retention
  terms are a procurement precondition, not a configuration preference.

## 8.6 AI-specific controls

Restating the load-bearing points, because this is where the audit questions will land:

1. The AI service has **no database credentials**, no S3 credentials and no tool access.
   It is a pure function from payload to result.
2. All retrieval context is assembled by the Core API under the caller's `AccessScope`.
   The embeddings table has one repository function, and it takes an `AccessScope`, not an
   org id.
3. Untrusted content (customer messages, uploaded documents) is delimited and explicitly
   labeled as data in prompts. Instructions found inside it are content to be summarized,
   not directives.
4. Outputs are schema-validated. A model that emits an instruction instead of a value
   fails validation and is discarded.
5. AI can be disabled per organization with one setting. The platform remains fully
   functional — this is testable, and it is tested.
6. Every model call is attributable: user, org, task, prompt version, model, tokens, cost.

## 8.7 Application hardening

- Input validation at the DTO boundary (`validator`), with length ceilings on every string
  and element ceilings on every array.
- Output encoding in React; user-supplied markdown is sanitized with a strict allow-list.
- CSP with nonces, `frame-ancestors 'none'`, `X-Content-Type-Options: nosniff`,
  `Referrer-Policy: strict-origin-when-cross-origin`, `Permissions-Policy` minimal.
- CSRF: `SameSite=Lax` cookies plus an origin check on all state-changing requests.
- Rate limits: per IP on auth endpoints, per user on writes, per org on AI tasks and exports.
- Response size and query-depth caps; `?expand=` limited to depth 1 ([doc 05](05-api-design.md) §5.3).
- Dependency scanning (`cargo audit`, `npm audit`, `pip-audit`) in CI, blocking on high
  severity; SBOM generated per release.
- No `unsafe` in first-party Rust crates (`#![forbid(unsafe_code)]`).

## 8.8 Audit trail

`events` is the audit log: actor, entity, before/after diff, request id, timestamp. It is
append-only — the application role holds `INSERT` and `SELECT` on it and no `UPDATE` or
`DELETE`; retention pruning runs as a separate maintenance role.

Administrative actions (role change, user suspension, template publication, rule
enablement, erasure, AI budget change) additionally raise an `admin.*` event and notify all
`owner`s. Audit records are queryable per entity in the UI, which is what makes "who
changed this project's due date" a five-second question.
