# Design: Principal-Based Cloud User and Token Management

Engram Cloud will move authentication and authorization from deployment-wide static secrets to first-class principals, hashed tokens, roles, and project grants. The design preserves all existing `/sync/*` route paths and payload contracts while replacing the internal auth decision with a principal-aware resolver.

The MVP implements human users first, keeps service accounts model-ready, and preserves `ENGRAM_CLOUD_TOKEN`, `ENGRAM_CLOUD_ADMIN`, and `ENGRAM_CLOUD_ALLOWED_PROJECTS` as bootstrap/legacy compatibility paths.

## Current-state anchors

| Area | Current implementation | Design impact |
| --- | --- | --- |
| Bearer auth | `internal/cloud/auth.Service.Authorize(*http.Request) error` compares the presented bearer token to `ENGRAM_CLOUD_TOKEN`. | Replace boolean auth with principal resolution while keeping a compatibility `Authorize` adapter for existing server wiring during migration. |
| Project auth | `auth.ProjectScopeAuthorizer` and `cloudserver.authorizeProjectScope` enforce global allowlists. | Add principal-aware grant enforcement and keep legacy allowlist behavior for env-token principals. |
| Sync routes | `internal/cloud/cloudserver` mounts `/sync/pull`, `/sync/pull/{chunkID}`, `/sync/push`, `/sync/mutations/push`, `/sync/mutations/pull`. | Keep route paths, methods, request bodies, and response bodies stable. |
| Mutation pull | `EnrolledProjectsProvider.EnrolledProjects() []string` filters pull results. | Return granted projects for managed principals; return current allowlist semantics for legacy principals. |
| Dashboard auth | Dashboard login accepts token, mints signed cookie containing token hash, and derives admin status from `ENGRAM_CLOUD_ADMIN`. | Store principal/session claims in the cookie and derive display/admin state from resolved principal role. |
| Persistence | `cloud_users` exists but is minimal; `cloud_sync_audit_log` already stores sync-related audit rows. | Extend schema instead of treating contributor analytics as managed users. |
| Dashboard `/admin/users` | Currently lists contributors, not managed accounts. | Repurpose or replace this admin surface to manage cloud principals/users, clearly separate from contributor browsing. |

## Architecture

### Package boundaries

| Package | Responsibility after this change |
| --- | --- |
| `internal/cloud/auth` | Principal domain types, token generation/hash/verification, bearer/dashboard session resolution, legacy env-principal compatibility, project grant authorization adapter. |
| `internal/cloud/cloudstore` | PostgreSQL persistence for principals, human users, token records, roles, project grants, and audit events. No HTTP or dashboard rendering logic. |
| `internal/cloud/cloudserver` | HTTP middleware, request context propagation, sync route authorization, admin JSON/form endpoints, and response status behavior. It depends on auth interfaces, not concrete token storage details. |
| `internal/cloud/dashboard` | Server-rendered admin UI and HTMX partials for managed users, tokens, grants, and audit browsing. It calls store/server-facing interfaces; it does not implement policy decisions. |
| `cmd/engram` | CLI bootstrap command and cloud runtime wiring from environment/config into cloudstore/auth/cloudserver. |
| `internal/cloud/remote` | No contract changes. It continues to send `Authorization: Bearer <token>` to existing sync endpoints. |

### Request flow

1. A client sends `Authorization: Bearer <token>` to an existing `/sync/*` route.
2. `cloudserver.withAuth` calls a principal resolver instead of only checking a static token.
3. The resolver returns a `Principal` with kind, role, enabled state, token identity, and legacy/managed source.
4. The server stores the principal in request context for the handler duration.
5. Push handlers call `AuthorizeProject(principal, project)` for each target project.
6. Pull handlers ask the authorizer for `EnrolledProjects(principal)` and pass that list to existing store queries.
7. Audit events include actor principal metadata and avoid storing raw token material.

Dashboard flow follows the same resolver: login validates a token, mints a signed session for the resolved principal, and all admin routes check `principal.role == admin`.

## Domain model

### Principal

```go
type Principal struct {
    ID          string
    Kind        PrincipalKind // human | service_account | legacy_sync | legacy_admin/bootstrap internal constants
    DisplayName string
    Role        Role          // admin | member for MVP
    Enabled     bool
    Source      PrincipalSource // managed_token | legacy_env_sync | legacy_env_admin | bootstrap_cli
    TokenID     string          // set for managed tokens
}
```

`principal.kind` is persisted as `human` or `service_account`. Legacy/bootstrap identities can be represented internally through `Source` and stable synthetic IDs, not as user-created rows unless a migration later chooses to materialize them.

### Roles

MVP roles are intentionally coarse:

| Role | Capability |
| --- | --- |
| `admin` | Dashboard/admin access, user lifecycle, token issue/revoke, grant create/revoke, bootstrap actions. |
| `member` | Sync access only for explicitly granted projects. No admin UI/API mutation access. |

Project grants carry project scope. Fine-grained project roles are deferred.

## Persistence design

The existing inline `CloudStore.migrate` should gain additive, non-destructive migrations. All tables should use stable textual IDs or BIGSERIAL IDs consistently with current `cloud_users`; implementation can prefer BIGSERIAL plus `id::text` compatibility to minimize existing code churn.

### Tables

#### `cloud_principals`

Stores the actor identity independent of token records.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | BIGSERIAL PRIMARY KEY | Principal identifier. |
| `kind` | TEXT NOT NULL | `human` or `service_account`; CHECK constraint recommended. |
| `display_name` | TEXT NOT NULL | Dashboard label. |
| `role` | TEXT NOT NULL DEFAULT `member` | `admin` or `member`; CHECK constraint recommended. |
| `enabled` | BOOLEAN NOT NULL DEFAULT TRUE | Disabled principals cannot authenticate. |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT NOW() | Creation time. |
| `updated_at` | TIMESTAMPTZ NOT NULL DEFAULT NOW() | Updated on lifecycle/role changes. |

Indexes:

- `idx_cloud_principals_kind`
- `idx_cloud_principals_role`
- `idx_cloud_principals_enabled`

#### `cloud_human_users`

Human-specific account profile. This avoids overloading existing contributor analytics and gives service accounts a clean future path.

| Column | Type | Notes |
| --- | --- | --- |
| `principal_id` | BIGINT PRIMARY KEY REFERENCES cloud_principals(id) ON DELETE CASCADE | One human profile per principal. |
| `username` | TEXT UNIQUE NOT NULL | CLI/dashboard-visible handle. |
| `email` | TEXT UNIQUE | Optional for self-hosted MVP; unique when present. |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT NOW() | Profile creation time. |

Existing `cloud_users` should not be used for contributor analytics going forward. If it is retained for backward compatibility, add a migration note and route new managed-user code to `cloud_principals` + `cloud_human_users`.

#### `cloud_principal_tokens`

Stores sync/admin login bearer tokens as hashes and metadata.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | BIGSERIAL PRIMARY KEY | Token record ID. |
| `principal_id` | BIGINT NOT NULL REFERENCES cloud_principals(id) ON DELETE CASCADE | Owner principal. |
| `token_prefix` | TEXT NOT NULL | Public token prefix for lookup/display, e.g. `egc_live_abc123`. |
| `token_hash` | TEXT NOT NULL UNIQUE | HMAC-SHA256 or Argon2id hash of token secret. Raw token is never stored. |
| `name` | TEXT NOT NULL DEFAULT '' | Operator-provided label. |
| `created_by_principal_id` | BIGINT REFERENCES cloud_principals(id) | Admin actor, nullable for bootstrap. |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT NOW() | Creation time. |
| `last_used_at` | TIMESTAMPTZ | Updated after successful auth; best-effort. |
| `revoked_at` | TIMESTAMPTZ | Non-null means token cannot authenticate. |
| `revoked_by_principal_id` | BIGINT REFERENCES cloud_principals(id) | Admin actor. |
| `revocation_reason` | TEXT | Optional reason. |

Indexes:

- Unique `token_hash`
- Index `token_prefix`
- Index `(principal_id, revoked_at)`

#### `cloud_project_grants`

Deny-by-default allowlist for managed principals.

| Column | Type | Notes |
| --- | --- | --- |
| `principal_id` | BIGINT NOT NULL REFERENCES cloud_principals(id) ON DELETE CASCADE | Grantee. |
| `project` | TEXT NOT NULL | Normalized project name. |
| `granted_by_principal_id` | BIGINT REFERENCES cloud_principals(id) | Admin actor. |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT NOW() | Grant time. |

Primary key: `(principal_id, project)`.

No wildcard grants for managed principals in MVP unless product explicitly chooses it later. Legacy env sync can still use existing wildcard behavior through `ENGRAM_CLOUD_ALLOWED_PROJECTS`.

#### `cloud_auth_audit_log`

Preferred long-term audit table for auth/admin events. Existing `cloud_sync_audit_log` can remain for sync pause rejections. If implementation wants one table, it may extend `cloud_sync_audit_log`, but the design favors a separate auth-focused table to avoid muddying contributor/project sync audit semantics.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | BIGSERIAL PRIMARY KEY | Audit row ID. |
| `occurred_at` | TIMESTAMPTZ NOT NULL DEFAULT NOW() | Event time. |
| `actor_principal_id` | BIGINT | Acting principal, nullable for legacy/bootstrap. |
| `actor_source` | TEXT NOT NULL | `managed`, `legacy_env_admin`, `legacy_env_sync`, `cli_bootstrap`, `system`. |
| `target_principal_id` | BIGINT | User/token/grant target when applicable. |
| `project` | TEXT | Grant/sync project when applicable. |
| `action` | TEXT NOT NULL | Enumerated action name. |
| `outcome` | TEXT NOT NULL | `success`, `denied`, `failed`. |
| `reason_code` | TEXT | Machine-readable reason. |
| `metadata` | JSONB | Redacted event details. Never raw tokens. |

Required MVP action names:

- `token.create`
- `token.revoke`
- `user.create`
- `user.enable`
- `user.disable`
- `grant.create`
- `grant.revoke`
- `admin.login`
- `bootstrap.dashboard`
- `bootstrap.cli`

## Principal resolver API

`internal/cloud/auth` should expose small interfaces so `cloudserver` and tests can use fakes without importing storage internals.

```go
type PrincipalResolver interface {
    ResolveBearerToken(ctx context.Context, token string) (Principal, error)
    MintDashboardSession(ctx context.Context, principal Principal) (string, error)
    ResolveDashboardSession(ctx context.Context, sessionToken string) (Principal, error)
}

type PrincipalProjectAuthorizer interface {
    AuthorizeProject(ctx context.Context, principal Principal, project string) error
    EnrolledProjects(ctx context.Context, principal Principal) ([]string, error)
}
```

Compatibility adapters:

- Keep `Authorize(*http.Request) error` during migration by resolving and discarding the principal, but new `cloudserver` code should use `ResolveBearerToken` so handlers can access the actor.
- Keep `ProjectAuthorizer` compatibility for legacy tests by wrapping a request-scoped principal-aware authorizer or introducing a `withPrincipalAuth` path for sync handlers.

Recommended request context helpers:

```go
func WithPrincipal(ctx context.Context, p Principal) context.Context
func PrincipalFromContext(ctx context.Context) (Principal, bool)
```

Only server/middleware should write principal context. Business logic should receive `Principal` explicitly where practical.

## Token generation, prefix, and hashing

### Token format

Use a recognizably Engram cloud token format:

```text
egc_<environment>_<prefix>_<secret>
```

Examples:

- `egc_live_ab12cd34_<random-secret>`
- `egc_dev_ab12cd34_<random-secret>`

MVP can use `egc_live_` only if environment distinction is not already modeled. The important split is:

- `token_prefix`: short non-secret identifier stored/displayed for admin lookup.
- `secret`: high-entropy random bytes shown once at creation.

Generation rules:

- Use `crypto/rand`.
- At least 32 bytes of raw entropy for the secret portion.
- Encode URL-safe without padding.
- Store only `token_prefix`, hash, metadata, and timestamps.
- Never log full raw tokens.
- Admin UI/API returns the raw token only in the creation response/page.

### Hash model

Use one of these, in order of preference:

1. HMAC-SHA256 with a dedicated cloud token pepper/secret for fast exact token lookup.
2. Argon2id if lookup uses prefix first and verifies a small candidate set.

HMAC-SHA256 is the lowest-risk MVP path because it supports deterministic lookup, constant-time comparison, and avoids storing raw credentials. The token pepper MUST be separate from the dashboard/session signing secret (`ENGRAM_JWT_SECRET`) so session-secret rotation does not implicitly rotate or invalidate stored token verifiers. If a deployment lacks the dedicated token pepper during migration, startup should fail clearly or use an explicitly documented one-time migration fallback rather than silently coupling token hashes to the session secret.

Hash input should include a domain separator:

```text
engram-cloud-token:v1:<raw-token>
```

Use `hmac.Equal` for verification.

## Auth middleware changes

### Sync routes

Replace `withAuth` internals with:

1. Parse `Authorization: Bearer <token>` exactly as today.
2. Resolve token to principal.
3. Reject if token is revoked, principal disabled, malformed, or unknown.
4. Store principal in request context.
5. Call the existing sync handler.

HTTP behavior:

| Case | Status | Body contract |
| --- | --- | --- |
| Missing/malformed/invalid/revoked token | `401 Unauthorized` | May keep current `http.Error` text; avoid leaking which condition matched. |
| Valid principal lacking project grant | `403 Forbidden` | Existing structured policy envelope from `writeActionableError` should remain. |
| Valid legacy env token outside `ENGRAM_CLOUD_ALLOWED_PROJECTS` | `403 Forbidden` | Preserve existing allowlist behavior. |

### Project authorization

Managed principals are deny-by-default:

- Push chunk: authorize the request project before storing.
- Mutation push: authorize every distinct project in the batch; reject all-or-nothing if any project is denied.
- Pull manifest/chunk: authorize requested project.
- Mutation pull: filter by `cloud_project_grants` for managed principal; empty grants returns an empty mutation list, not all projects.

Legacy env sync principal keeps current `ENGRAM_CLOUD_ALLOWED_PROJECTS` semantics:

- `*` means all normalized projects.
- Empty allowlist remains invalid/blocked as it is today.
- Normalization continues through `store.NormalizeProject`.

### Dashboard/session auth

Dashboard sessions should no longer need to store a raw bearer token hash that is re-compared against env tokens only. The signed session payload should contain:

```json
{
  "principal_id": "123",
  "principal_source": "managed_token",
  "role": "admin",
  "display_name": "Alice",
  "iat": 123,
  "exp": 456,
  "session_version": 1
}
```

Resolution rules:

- For managed principals, re-check principal enabled state and role from authoritative storage on every protected dashboard/admin request. Signed cookie claims are hints for lookup/display only; stale claims must not preserve access after disablement or demotion.
- Set dashboard auth cookies with `HttpOnly`, `Secure` when the request is HTTPS or production config requires it, and `SameSite=Lax` at minimum. Mutation routes should keep existing CSRF-safe form semantics and may add stronger CSRF tokens separately.
- For legacy admin sessions, encode `principal_source = legacy_env_admin`; after a managed admin exists, treat this as explicit bootstrap/recovery access rather than a silent permanent bypass around managed admin roles.
- Dashboard `IsAdmin` should read the resolved principal role/source, not compare the original token to `ENGRAM_CLOUD_ADMIN` directly.
- Dashboard `GetDisplayName` should return principal display name and keep `OPERATOR` fallback for legacy/bootstrap.

## Admin API and dashboard surface

MVP can implement admin actions as server-rendered forms with HTMX enhancements, matching the existing dashboard style. JSON endpoints are optional unless needed by CLI or tests; form POSTs must still work as normal HTTP posts.

### Routes

Recommended dashboard/admin routes:

| Route | Method | Purpose |
| --- | --- | --- |
| `/dashboard/admin/users` | GET | Managed user shell. Must not list contributor analytics. |
| `/dashboard/admin/users/list` | GET | HTMX partial of managed human users. |
| `/dashboard/admin/users/create` | POST | Create human user with role. |
| `/dashboard/admin/users/{id}/enable` | POST | Enable user/principal. |
| `/dashboard/admin/users/{id}/disable` | POST | Disable user/principal. |
| `/dashboard/admin/users/{id}/tokens` | GET | Token metadata for a user. |
| `/dashboard/admin/users/{id}/tokens/create` | POST | Create token; show raw token once. |
| `/dashboard/admin/tokens/{id}/revoke` | POST | Revoke token. |
| `/dashboard/admin/users/{id}/grants` | GET | Project grants for a user. |
| `/dashboard/admin/users/{id}/grants/create` | POST | Grant project. |
| `/dashboard/admin/users/{id}/grants/{project}/revoke` | POST | Revoke project grant. |

All mutation routes are admin-only and must create audit events.

### UI rules

- Separate navigation labels: `Managed Users` for accounts, `Contributors` for observed synced contributors.
- Token creation page must display the raw token once with copy guidance and a warning that it cannot be retrieved again.
- Token list displays name, prefix, created time, last-used time, revoked state, and revoke action.
- Grant list displays normalized projects and revoke action near each grant.
- Empty states explain deny-by-default: a managed user with no grants cannot sync any project.
- Existing HTMX partial endpoints should remain meaningful HTML fragments and forms should degrade to normal POST/redirect behavior.

## CLI bootstrap command

Add a headless bootstrap path under the existing `engram cloud` command tree.

Recommended shape:

```bash
engram cloud bootstrap admin \
  --username <name> \
  [--email <email>] \
  [--role admin] \
  [--grant-project <project>]... \
  [--issue-token <token-name>] \
  [--dsn <postgres-dsn>]
```

Operational behavior:

- Uses cloud runtime DB configuration from env by default (`cloud.ConfigFromEnv()`), with `--dsn` only if the project already uses CLI DSN override conventions.
- Creates the first managed human admin when no managed admin exists.
- If a managed admin already exists, fail with a clear message unless the operator invokes an explicitly named recovery command/path with legacy admin auth evidence. Do not create additional admins silently.
- Can optionally issue an initial sync token and print it once.
- Can optionally grant projects while preserving normalized project rules.
- Writes `bootstrap.cli` audit events.
- Never writes raw tokens to database or logs.
- Must not disable, demote, or invalidate the last usable managed admin path through routine commands; any recovery override must be explicit, documented, and audited.

Dashboard bootstrap should offer equivalent first-admin creation when authenticated by `ENGRAM_CLOUD_ADMIN` and no managed admin exists, writing `bootstrap.dashboard` audit events.

## Legacy compatibility

### Legacy sync token

`ENGRAM_CLOUD_TOKEN` continues to authorize existing clients exactly as today externally. Internally it resolves to a synthetic principal:

| Field | Value |
| --- | --- |
| `ID` | `legacy:sync` |
| `Kind` | internal legacy/bootstrap kind, not persisted as `service_account` in MVP |
| `DisplayName` | `LEGACY_SYNC` |
| `Role` | `member` |
| `Source` | `legacy_env_sync` |

Project authorization uses `ENGRAM_CLOUD_ALLOWED_PROJECTS` exactly as the current `ProjectScopeAuthorizer` does.

### Legacy admin token

`ENGRAM_CLOUD_ADMIN` continues to authenticate dashboard bootstrap access and explicit recovery operations. It must not become an undocumented permanent bypass around managed admin roles after managed admins exist. Internally it resolves to:

| Field | Value |
| --- | --- |
| `ID` | `legacy:admin` |
| `DisplayName` | `OPERATOR` or `LEGACY_ADMIN` |
| `Role` | `admin` |
| `Source` | `legacy_env_admin` |

Legacy admin is a migration aid. Before a managed admin exists, it can create the first managed admin. After a managed admin exists, normal admin operations should require a managed admin principal; legacy admin use should be limited to explicit, audited recovery paths. Docs should describe managed admins as the long-term model.

### Insecure mode

`ENGRAM_CLOUD_INSECURE_NO_AUTH=1` should remain development-only and must not implicitly create a managed principal. If supported for dashboard during tests/dev, it should resolve to a clearly marked `Source = insecure_dev` principal and never be documented as production bootstrap.

## Audit model

All required MVP audit events should be emitted synchronously with admin mutations. If audit insertion fails for an admin mutation, prefer failing the mutation rather than silently losing an authoritative security event. For sync rejection audit events that already exist, preserve current best-effort behavior unless implementation chooses to harden it in a separate spec.

Event metadata examples:

| Action | Metadata |
| --- | --- |
| `token.create` | `{ "token_id": "...", "token_prefix": "...", "token_name": "..." }` |
| `token.revoke` | `{ "token_id": "...", "token_prefix": "...", "reason": "..." }` |
| `user.create` | `{ "role": "admin", "kind": "human" }` |
| `user.disable` | `{ "previous_enabled": true }` |
| `grant.create` | `{ "project": "normalized-project" }` |
| `admin.login` | `{ "source": "managed_token" }` |
| `bootstrap.cli` | `{ "created_admin": true, "issued_token": true }` |

Do not store raw token values, request authorization headers, or dashboard cookie values in audit metadata.

## Migration and rollout

### Migration steps

1. Add new tables/indexes with `CREATE TABLE IF NOT EXISTS` and `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` patterns matching current cloudstore migration style.
2. Keep existing `cloud_users`, `cloud_chunks`, `cloud_mutations`, `cloud_project_controls`, and `cloud_sync_audit_log` intact.
3. Wire resolver to check managed token storage first, then legacy env tokens.
4. Add dashboard/CLI bootstrap paths for the first managed admin.
5. Add grant enforcement for managed principals while preserving legacy allowlist enforcement for env sync principal.
6. Update docs to explain migration from env tokens to managed users/tokens.

### Rollback

Rollback is operational, not destructive:

- Leave new tables in place.
- Reconfigure clients to use existing `ENGRAM_CLOUD_TOKEN`.
- Keep `ENGRAM_CLOUD_ADMIN` for dashboard/admin access.
- Disable or stop issuing managed tokens if needed.
- Because sync payloads and storage tables are unchanged, cloud memory data remains readable by the legacy auth path.

Do not include migrations that remove env-token support or mutate existing sync payload tables in this MVP.

## Testing strategy

### Unit tests

| Package | Coverage |
| --- | --- |
| `internal/cloud/auth` | Token generation entropy/format, hash verification, invalid/revoked token rejection, legacy env token resolution, dashboard session encode/decode, role checks. |
| `internal/cloud/cloudstore` | Principal CRUD, user create/enable/disable, token create/list/revoke, grant create/revoke/list, deny-by-default queries, audit insert/list. |
| `cmd/engram` | CLI bootstrap argument parsing, first-admin guard, raw token printed once, duplicate bootstrap refusal. |

### Handler tests

| Area | Coverage |
| --- | --- |
| Sync auth | Existing `/sync/*` routes still accept valid legacy token and reject invalid tokens. |
| Managed token sync | Managed token can pull/push only granted projects. Missing grant returns 403. |
| Mutation push | Batch with any unauthorized project rejects the entire batch. |
| Mutation pull | Managed principal with grants sees only granted projects; no grants returns empty list. |
| Revocation | Revoked token fails subsequent sync/admin requests. |
| Dashboard login | Managed admin login creates a dashboard session; member cannot access admin routes. |
| Admin routes | Create user, disable/enable user, create/revoke token, create/revoke grant all require admin and emit audit. |
| Bootstrap | Legacy admin token can create first managed admin; CLI can create first managed admin; no lockout when no managed admin exists. |

### Regression tests

- `/sync/*` route methods, paths, request payloads, and response schemas remain unchanged.
- `ENGRAM_CLOUD_ALLOWED_PROJECTS=*`, explicit project list, and empty/missing allowlist preserve current legacy behavior.
- Dashboard `/dashboard/contributors` continues to show contributor analytics and is not replaced by managed users.
- Raw tokens do not appear in stored rows, logs, or audit metadata.

### Docs verification

Update docs in the implementation PR for:

- Managed user/token concepts.
- First admin bootstrap via dashboard and CLI.
- Token creation and one-time display.
- Project grants and deny-by-default behavior.
- Legacy env-token migration path.
- Current compatibility of existing sync clients.

## Implementation notes and sequencing

A safe implementation order is:

1. Add storage tables and store methods with tests.
2. Add auth principal resolver and token hashing/generation tests.
3. Wire cloudserver middleware to carry principals while keeping legacy adapters.
4. Enforce project grants for managed principals and preserve legacy allowlist behavior.
5. Add CLI/dashboard bootstrap for first admin.
6. Add admin user/token/grant dashboard flows.
7. Add audit events and dashboard audit visibility updates.
8. Update docs and migration guidance.

This sequence keeps sync compatibility testable at every step and avoids a rollout where managed tokens exist but project grants are not enforced.
