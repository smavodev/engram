# Apply Progress: Cloud User Token Management

## Current branch

`feat/cloud-user-token-management-server-sync`

## Chain context

- Tracker branch: `feat/cloud-user-token-management`
- Chain strategy: `feature-branch-chain`
- Current slice: PR 2 — server middleware and sync grant enforcement
- Prior slice on this branch: PR 1A — `internal/cloud/auth` principal and token foundation, committed as `d4c3b38 feat(cloud): add auth principal foundation`
- Out of scope for this slice: dashboard admin/user-management, CLI bootstrap, docs beyond this progress file and task checkboxes

## Structured status consumed / produced before apply

```yaml
schemaName: spec-driven
changeName: cloud-user-token-management
artifactStore: openspec
planningHome:
  root: /Users/alanbuscaglia/work/engram/openspec
  changesDir: /Users/alanbuscaglia/work/engram/openspec/changes
changeRoot: /Users/alanbuscaglia/work/engram/openspec/changes/cloud-user-token-management
artifactPaths:
  proposal: [openspec/changes/cloud-user-token-management/proposal.md]
  specs: [openspec/changes/cloud-user-token-management/spec.md]
  design: [openspec/changes/cloud-user-token-management/design.md]
  tasks: [openspec/changes/cloud-user-token-management/tasks.md]
  applyProgress: [openspec/changes/cloud-user-token-management/apply-progress.md]
contextFiles:
  proposal: [openspec/changes/cloud-user-token-management/proposal.md]
  specs: [openspec/changes/cloud-user-token-management/spec.md]
  design: [openspec/changes/cloud-user-token-management/design.md]
  tasks: [openspec/changes/cloud-user-token-management/tasks.md]
  applyProgress: [openspec/changes/cloud-user-token-management/apply-progress.md]
artifacts:
  proposal: done
  specs: done
  design: done
  tasks: done
  applyProgress: done
applyState: ready
dependencies:
  apply: ready
  verify: ready
  sync: blocked
  archive: blocked
actionContext:
  mode: repo-local
  workspaceRoot: /Users/alanbuscaglia/work/engram
  allowedEditRoots: [/Users/alanbuscaglia/work/engram]
  warnings: []
nextRecommended: apply PR2 server middleware/sync grant enforcement after PR1B review
isNonAuthoritative: false
```

## Progress

### Previously completed: PR 1A auth foundation

- Added `internal/cloud/auth/foundation_test.go` covering:
  - principal kind, role, and source validation/string values;
  - managed token format and entropy shape;
  - dedicated token pepper requirement;
  - domain-separated HMAC token verifier behavior;
  - no raw token material in token verifiers;
  - resolver rejection for revoked tokens and disabled principals;
  - legacy env sync/admin principal resolution.
- Added `internal/cloud/auth/foundation.go` with:
  - principal domain types;
  - MVP roles and principal sources;
  - managed token generation;
  - dedicated pepper HMAC token hasher/verifier;
  - storage-agnostic managed token lookup interface;
  - principal resolver with managed-token and legacy-env resolution.
- Updated `tasks.md` to mark the auth RED/GREEN tasks complete.

### Completed in PR 1B storage/cloudstore foundation

- Added `internal/cloud/cloudstore/identity_storage_test.go` with Postgres-gated integration tests for:
  - additive migration creation of `cloud_principals`, `cloud_human_users`, `cloud_principal_tokens`, `cloud_project_grants`, and `cloud_auth_audit_log`;
  - preservation of existing `cloud_chunks` and `cloud_mutations` rows across a second migration run;
  - principal create/get/update lifecycle;
  - human user create/list/disable lifecycle;
  - token metadata create/list/revoke with list responses omitting token hash/raw token material and database persistence using hash-only verifier values;
  - project grant create/list/revoke with normalized duplicate handling;
  - active-admin existence checks and last-active-admin guard helper;
  - auth audit insert/list with non-secret metadata;
  - error paths for invalid principal kind/role, duplicate usernames/emails, duplicate token hashes, and empty projects.
- Extended `internal/cloud/cloudstore/cloudstore.go` migrations additively with the five auth foundation tables and related indexes.
- Added `internal/cloud/cloudstore/identity.go` with storage-only cloudstore types and methods for principals, managed human users, token metadata, project grants, admin checks, and auth audit events.
- Kept the slice storage-only: no cloudserver, dashboard, CLI bootstrap, docs, or legacy env-token behavior changes.
- Removed the local `.codegraph/` index generated during structural inspection so no generated/local files remain.
- Updated persisted `tasks.md` checkboxes for completed PR 1 / PR1B tasks.


### Completed in PR 2 server middleware and sync grant enforcement

- Added `internal/cloud/cloudserver/principal_auth_test.go` covering:
  - existing `/sync/*` routes accepting valid legacy/principal-resolved tokens;
  - malformed, unknown, and revoked managed-token requests returning the existing `unauthorized:` 401 style;
  - principal propagation into request context;
  - legacy `auth.Service` resolving `ENGRAM_CLOUD_TOKEN` as a legacy sync principal;
  - managed principal grant enforcement for chunk pull and chunk push;
  - managed mutation push rejecting mixed granted/ungranted batches all-or-nothing;
  - managed mutation pull returning only granted project mutations.
- Updated `internal/cloud/auth/auth.go` so the existing legacy service implements `ResolveBearerToken`, allowing cloudserver to use one principal-aware path while preserving `Authorize` compatibility.
- Updated `internal/cloud/cloudserver/cloudserver.go` with:
  - principal resolver detection;
  - principal request-context helpers;
  - `WithPrincipalProjectAuthorizer` for managed principal project grants;
  - shared auth middleware that resolves principals when available and falls back to existing `Authorize` behavior;
  - principal-aware project authorization for chunk manifest/pull/push routes.
- Updated `internal/cloud/cloudserver/mutations.go` to use principal-aware project grants for mutation push and mutation pull filtering, while preserving legacy project authorizer behavior.
- Kept this slice server/sync-only: no dashboard admin, CLI bootstrap, or managed-user UI routes.

### PR 2 validation

```bash
go test ./internal/cloud/cloudserver
go test ./...
```

Results:

- `go test ./internal/cloud/cloudserver`: PASS.
- `go test ./...`: PASS.

### PR 2 review remediation

- Principal project authorization now fails closed when `WithPrincipalProjectAuthorizer` is configured but no principal is present in request context.
- Mutation pull now returns a policy error instead of using an unfiltered `nil` project list when a principal authorizer is configured without a resolved principal.
- Managed principals with no grants now get an explicit empty project filter for mutation pull, preventing nil-as-all leakage.
- Legacy env sync principals continue to use legacy `ENGRAM_CLOUD_ALLOWED_PROJECTS` semantics even when a principal project authorizer is configured for managed principals.
- Added fail-closed regression coverage for miswired principal project authorization.
- Added explicit managed granted chunk push and granted/denied chunk pull route coverage.

### PR 2 note

The first broad PR2 subagent timed out after writing a small, reviewable diff. The parent recovered the partial work, ran `gofmt`, targeted cloudserver tests, and the full test suite. Review still required before commit.

## Persisted task checkbox updates

The following task lines are now visibly checked in `openspec/changes/cloud-user-token-management/tasks.md`:

- [x] RED: Add cloudstore migration tests in `internal/cloud/cloudstore/` proving additive creation of `cloud_principals`, `cloud_human_users`, `cloud_principal_tokens`, `cloud_project_grants`, and `cloud_auth_audit_log` without altering existing sync tables.
- [x] GREEN: Extend `internal/cloud/cloudstore/cloudstore.go` migrations and add focused store methods for principal CRUD, human user create/list/enable/disable, token metadata create/list/revoke, project grant create/list/revoke, admin existence checks, and auth audit insertion.
- [x] TRIANGULATE: Add error-path tests for duplicate usernames/emails, invalid roles/kinds, duplicate grants, revoked tokens, missing pepper, and hash-only persistence.
- [x] REFACTOR: Keep storage interfaces small so `internal/cloud/cloudserver` can depend on auth/store contracts without importing dashboard rendering logic.
- [x] Verify: `go test ./internal/cloud/auth ./internal/cloud/cloudstore` and `go test ./...`.
- [x] Rollback boundary: revert new migrations and auth foundation only; legacy env-token sync remains untouched.

## TDD Cycle Evidence

| Task | Test File | Layer | Safety Net | RED | GREEN | TRIANGULATE | REFACTOR |
|------|-----------|-------|------------|-----|-------|-------------|----------|
| PR1B migration foundation | `internal/cloud/cloudstore/identity_storage_test.go` | Integration, Postgres-gated | ✅ `go test ./internal/cloud/cloudstore ./internal/cloud/auth` passed before production edits | ✅ Compile failed on missing storage API/migrations after tests were written | ✅ `go test ./internal/cloud/cloudstore -run 'TestAuthFoundationMigrationsAreAdditive|TestCloudstorePrincipalHumanTokenGrantAndAuditLifecycle|TestCloudstoreIdentityGuardsAndErrorPaths'` passed/skipped cleanly without DSN | ✅ Additive migration preservation test included existing sync rows and second migration run | ✅ Migration statements are additive `CREATE ... IF NOT EXISTS` / indexes only |
| PR1B storage methods | `internal/cloud/cloudstore/identity_storage_test.go` | Integration, Postgres-gated | ✅ Existing cloudstore/auth packages green | ✅ Compile failed on missing `CreatePrincipal`, `CreateHumanUser`, token/grant/admin/audit APIs | ✅ Focused cloudstore test target passed/skipped cleanly without DSN | ✅ Error-path coverage for invalid kind/role, duplicate username/email, duplicate token hash, duplicate grants, empty project, revocation metadata, and hash-only persistence | ✅ Storage-only implementation isolated in `identity.go`; no server/dashboard imports |

### Test Summary

- Total tests written: 3 Postgres-gated integration tests.
- Total tests passing: targeted package and full suite pass in this environment; the new Postgres-gated tests skip at runtime because `CLOUDSTORE_TEST_DSN` is not set.
- Layers used: Integration (3), Unit (0), E2E (0).
- Approval tests: None — this was additive storage behavior, not a behavior-preserving refactor.
- Pure functions created: small validation/normalization helpers in `identity.go`.

## Validation run

```bash
go test ./internal/cloud/cloudstore ./internal/cloud/auth
go test ./...
git diff --check
```

Results:

- `go test ./internal/cloud/cloudstore ./internal/cloud/auth`: PASS.
- `go test ./...`: PASS.
- `git diff --check`: PASS.

Review remediation after PR1B:

- Project grant normalization now canonicalizes whitespace and punctuation to stable grant slugs, so `Alpha Project`, `alpha project`, and `alpha-project` map to the same project grant key.
- Last-active-admin protection moved into storage mutation paths for principal update and human enable/disable, with transaction-level checks returning `ErrLastActiveAdmin`; the guard now uses a transaction-scoped advisory lock so concurrent admin removals serialize.
- Auth audit metadata now rejects sensitive keys such as raw tokens, authorization headers, cookies, token hashes, passwords, and bearer values while still allowing safe `token_prefix` metadata; nested maps, typed maps, arrays, and slices are inspected.
- Added non-Postgres pure helper tests for project grant normalization and sensitive audit metadata classification so important storage-adjacent contracts execute even when `CLOUDSTORE_TEST_DSN` is unset.
- Added a Postgres-gated concurrent last-admin removal regression test for DSN-backed runs.

Additional RED/GREEN detail:

- RED command: `go test ./internal/cloud/cloudstore -run 'TestAuthFoundationMigrationsAreAdditive|TestCloudstorePrincipalHumanTokenGrantAndAuditLifecycle|TestCloudstoreIdentityGuardsAndErrorPaths'`
- RED result: compile failure because `CloudStore` did not yet expose the PR1B storage methods/types.
- GREEN result after implementation: PASS, with the new integration tests skipped because `CLOUDSTORE_TEST_DSN` is not configured locally.

## Files changed

- `internal/cloud/cloudstore/cloudstore.go`
- `internal/cloud/cloudstore/identity.go`
- `internal/cloud/cloudstore/identity_storage_test.go`
- `openspec/changes/cloud-user-token-management/tasks.md`
- `openspec/changes/cloud-user-token-management/apply-progress.md`

## Changed-line estimate

- Code/test slice estimate before this progress update: ~911 added lines plus 6 task-line checkbox edits.
- This exceeds the preferred ~700-line review target, but remains storage-only and does not expand into server wiring. Treat PR1B as a size-risk review item or split storage API/tests if the maintainer wants a tighter diff before commit/PR.

## Deviations from design

- No cloudserver/auth wiring was added in PR1B, by design for this slice.
- `internal/cloud/cloudstore` cannot directly import `internal/cloud/auth` types because existing `internal/cloud/auth` already imports `cloudstore`; storage types are therefore local storage DTOs for now. A later server/auth wiring slice should add adapters without introducing an import cycle.
- Postgres integration assertions are present but skipped locally without `CLOUDSTORE_TEST_DSN`; a Postgres-backed CI/local run should execute them before PR merge.

## Remaining tasks

Exact unchecked task lines remaining in `tasks.md`:

```markdown
- [ ] RED: Add handler tests in `internal/cloud/cloudserver/` proving existing `/sync/*` routes still accept valid legacy tokens and reject invalid/malformed/revoked managed tokens with current auth error style.
- [ ] GREEN: Replace `withAuth` internals in `internal/cloud/cloudserver/cloudserver.go` with principal resolution, request context helpers, and compatibility adapters for existing auth callers.
- [ ] RED: Add push/pull authorization tests for managed principals: granted project succeeds, ungranted project returns `403`, mutation batch with any ungranted project rejects all-or-nothing, mutation pull leaks no ungranted projects.
- [ ] GREEN: Wire principal-aware project authorization through sync chunk and mutation handlers, including `internal/cloud/cloudserver/mutations.go`, while preserving legacy `ENGRAM_CLOUD_ALLOWED_PROJECTS` wildcard/list/empty semantics.
- [ ] TRIANGULATE: Add regression tests for legacy sync principal behavior under `ENGRAM_CLOUD_ALLOWED_PROJECTS=*`, explicit lists, normalized projects, and empty/missing allowlist.
- [ ] REFACTOR: Keep sync payload structs and route registration unchanged; auth changes must be internal only.
- [ ] Verify: targeted cloudserver sync tests and `go test ./...`.
- [ ] Rollback boundary: disable managed-token resolver wiring and retain legacy env-token authorization path.
- [ ] RED: Add admin authorization tests in `internal/cloud/cloudserver/` proving only managed admin principals can create users, issue/revoke tokens, and create/revoke grants; members receive forbidden responses and no state changes.
- [ ] GREEN: Add admin form/API handlers under `internal/cloud/cloudserver/` for human user create/list/enable/disable, token create/list/revoke, and grant create/list/revoke, backed by cloudstore methods.
- [ ] RED: Add dashboard-session tests proving managed admin login succeeds, member admin access fails, disabled/demoted users lose access on the next protected request, and secure cookie attributes are set correctly.
- [ ] GREEN: Update dashboard auth/session handling in `internal/cloud/cloudserver` and `internal/cloud/dashboard` so signed cookies carry principal claims but every protected request revalidates enabled state and role from storage.
- [ ] RED: Add bootstrap tests for legacy dashboard/admin credential creating the first managed admin, rejecting duplicate first-admin bootstrap, and preventing accidental removal of the last usable managed admin path.
- [ ] GREEN: Implement dashboard bootstrap route/handler and last-admin protections, treating `ENGRAM_CLOUD_ADMIN` as explicit bootstrap/recovery access after managed admins exist.
- [ ] RED: Add audit tests for token create/revoke, user create/enable/disable, grant create/revoke, admin login, dashboard bootstrap, accepted/rejected legacy recovery actions, and redaction of raw tokens.
- [ ] GREEN: Emit synchronous `cloud_auth_audit_log` events for admin/security mutations; fail authoritative admin mutations if audit insertion fails.
- [ ] Verify: targeted admin/dashboard/bootstrap tests and `go test ./...`.
- [ ] Rollback boundary: remove admin/bootstrap routes while retaining storage/auth foundation and legacy auth behavior.
- [ ] RED: Add dashboard rendering/handler tests for `/dashboard/admin/users`, `/dashboard/admin/users/list`, token partials, grant partials, and contributor/managed-user separation.
- [ ] GREEN: Update `internal/cloud/dashboard/dashboard.go` and related templ/templates/assets to show `Managed Users` separately from contributor analytics.
- [ ] GREEN: Add server-rendered forms and HTMX-compatible partials for user create, enable/disable, token create/show-once, token revoke, grant create, and grant revoke.
- [ ] TRIANGULATE: Test non-HTMX form POST/redirect behavior and HTMX partial responses; partials must be meaningful HTML without hidden client-side policy logic.
- [ ] TRIANGULATE: Test empty states explaining deny-by-default project grants and token show-once warnings.
- [ ] REFACTOR: Keep policy checks in server/auth/store layers; dashboard code must render outcomes, not make authorization decisions.
- [ ] Verify: dashboard package tests plus `go test ./...`.
- [ ] Rollback boundary: remove dashboard UI routes/templates without affecting already-tested admin handlers.
- [ ] RED: Add CLI tests in `cmd/engram/` for `engram cloud bootstrap admin --username ...`, duplicate bootstrap refusal, optional token issuance printed once, optional project grants, invalid input, and audit event creation.
- [ ] GREEN: Implement `engram cloud bootstrap admin` in `cmd/engram/cloud.go`, using cloud runtime DB configuration by default and an existing DSN override convention only if already present.
- [ ] TRIANGULATE: Test that raw managed tokens are never persisted, logged, audited, rendered in token metadata lists, or printed except the creation/bootstrap response.
- [ ] GREEN: Update docs discovery targets affected by cloud setup and sync auth, starting with `README.md`, `docs/`, `CONTRIBUTING.md`, and any cloud deployment docs found by `rg "ENGRAM_CLOUD_TOKEN|ENGRAM_CLOUD_ADMIN|ENGRAM_CLOUD_ALLOWED_PROJECTS|cloud bootstrap"`.
- [ ] GREEN: Document managed users/tokens, dedicated token pepper, first-admin dashboard bootstrap, CLI bootstrap, project grants, deny-by-default managed principals, legacy env-token migration, and rollback to legacy sync credentials.
- [ ] RED: Add regression tests that `/sync/*` route methods, paths, request schemas, and response schemas remain unchanged for existing clients.
- [ ] GREEN: Fix any contract drift found by regression tests without changing MVP payloads.
- [ ] REFACTOR: Run `gofmt` on touched Go files and remove any temporary test seams not needed by production behavior.
- [ ] Verify: `go test ./...`, targeted cloud tests (`go test ./internal/cloud/... ./cmd/engram`), and `go test -cover ./...`.
- [ ] Rollback boundary: revert CLI/docs/audit hardening slice while keeping prior reviewed server behavior intact.
- [ ] Managed human users are distinct from contributor analytics.
- [ ] Managed tokens authenticate principals; authorization uses principal role and project grants.
- [ ] Token hashes use a dedicated cloud token pepper, not the dashboard/session signing secret.
- [ ] Raw token values are shown once and never stored or audited.
- [ ] Disabled users, revoked tokens, and revoked grants stop future access immediately.
- [ ] Legacy `ENGRAM_CLOUD_TOKEN`, `ENGRAM_CLOUD_ADMIN`, and `ENGRAM_CLOUD_ALLOWED_PROJECTS` behavior remains compatible during migration.
- [ ] Managed principals are deny-by-default for project sync.
- [ ] Dashboard cookies are `HttpOnly`, `SameSite=Lax` or stronger, and `Secure` under HTTPS/production rules.
- [ ] CLI and dashboard can create the first managed admin safely.
- [ ] Audit events cover all required MVP identity/security actions without secret leakage.
- [ ] Documentation matches real routes, commands, environment variables, and rollback behavior.
```

## Risks

- PR1B code/test diff is above the preferred ~700-line target. It remains bounded to storage-only files, but reviewers may still prefer a split before commit/PR.
- New cloudstore integration tests require `CLOUDSTORE_TEST_DSN` to execute against Postgres; in this environment they compile and skip.
- Storage DTOs currently duplicate some auth-domain string values to avoid the existing `auth -> cloudstore` import direction. The next auth/server wiring slice should be careful not to create a cycle.

---

## PR3A apply update — admin API handlers and audit-backed mutations

### Current branch

`feat/cloud-user-token-management-admin-bootstrap`

### Chain context

- Tracker branch: `feat/cloud-user-token-management`
- Parent branch for this feature-branch-chain slice: `feat/cloud-user-token-management-server-sync`
- Current slice: bounded PR3A from PR 3 — admin authorization and storage-backed admin API/form handlers only
- Prior committed slices supplied by parent context:
  - PR1A auth foundation: `d4c3b38 feat(cloud): add auth principal foundation`
  - PR1B storage foundation: `9defadf feat(cloud): add identity storage foundation`
  - PR2 server sync grant enforcement: `2669f3b feat(cloud): enforce principal sync grants`
- Out of scope for PR3A: dashboard session login/cookie revalidation, dashboard bootstrap first-admin flow, CLI bootstrap, dashboard rendered managed-user UX/templates, docs outside this progress/tasks update.

### Structured status consumed / produced before apply

```yaml
schemaName: spec-driven
changeName: cloud-user-token-management
artifactStore: openspec
planningHome:
  root: /Users/alanbuscaglia/work/engram/openspec
  changesDir: /Users/alanbuscaglia/work/engram/openspec/changes
changeRoot: /Users/alanbuscaglia/work/engram/openspec/changes/cloud-user-token-management
artifactPaths:
  proposal: [openspec/changes/cloud-user-token-management/proposal.md]
  specs: [openspec/changes/cloud-user-token-management/spec.md]
  design: [openspec/changes/cloud-user-token-management/design.md]
  tasks: [openspec/changes/cloud-user-token-management/tasks.md]
  applyProgress: [openspec/changes/cloud-user-token-management/apply-progress.md]
contextFiles:
  proposal: [openspec/changes/cloud-user-token-management/proposal.md]
  specs: [openspec/changes/cloud-user-token-management/spec.md]
  design: [openspec/changes/cloud-user-token-management/design.md]
  tasks: [openspec/changes/cloud-user-token-management/tasks.md]
  applyProgress: [openspec/changes/cloud-user-token-management/apply-progress.md]
artifacts:
  proposal: done
  specs: done
  design: done
  tasks: done
  applyProgress: done
taskProgress:
  total: 55
  complete: 18
  remaining: 37
applyState: ready
dependencies:
  apply: ready
  verify: ready
  sync: blocked
  archive: blocked
actionContext:
  mode: repo-local
  workspaceRoot: /Users/alanbuscaglia/work/engram
  allowedEditRoots: [/Users/alanbuscaglia/work/engram]
  warnings: []
nextRecommended: continue PR3 dashboard-session/bootstrap/admin-login audit tasks or run sdd-verify on the bounded PR3A slice
isNonAuthoritative: false
```

### Review workload / PR boundary

- `tasks.md` forecast has `400-line budget risk: High` and `Chained PRs recommended: Yes`.
- Parent provided a resolved bounded delivery path: feature-branch-chain PR3A, implementation limited to admin authorization and storage-backed admin API handlers.
- This slice stayed inside `internal/cloud/cloudserver/` plus OpenSpec progress/task updates and did not expand into dashboard UX/bootstrap/CLI.

### Completed in PR3A

- Added RED handler tests in `internal/cloud/cloudserver/admin_handlers_test.go` proving:
  - managed member principals receive `403` for user create/enable/disable, token create/revoke, and grant create/revoke;
  - legacy admin principals also receive `403`, proving the handlers require a managed admin principal, not just an admin-shaped legacy/bootstrap identity;
  - forbidden requests do not call mutation methods, do not create success audit events, and leave user/token/grant collections unchanged.
- Added GREEN admin API/form-level handlers in `internal/cloud/cloudserver/admin_handlers.go` and route registration/options in `internal/cloud/cloudserver/cloudserver.go` for:
  - `GET /admin/users`
  - `POST /admin/users`
  - `POST /admin/users/{principalID}/enable`
  - `POST /admin/users/{principalID}/disable`
  - `GET /admin/users/{principalID}/tokens`
  - `POST /admin/users/{principalID}/tokens`
  - `POST /admin/tokens/{tokenID}/revoke`
  - `GET /admin/users/{principalID}/grants`
  - `POST /admin/users/{principalID}/grants`
  - `POST /admin/users/{principalID}/grants/{project}/revoke`
- Added storage-backed handler boundary `AdminIdentityStore`, satisfied by existing cloudstore identity methods.
- Added `WithAdminIdentityStore` and `WithManagedTokenHasher` options for cloudserver wiring/tests.
- Added token issuance through `cloudauth.GenerateManagedToken("live")` and `ManagedTokenHasher.Hash`, returning raw token only in the token creation response.
- Added sanitized token metadata responses that omit hash/raw token fields from token metadata list/create metadata.
- Added RED/GREEN audit coverage for this PR3A slice: token create/revoke, user create/enable/disable, grant create/revoke, redacted audit metadata, and audit fail-closed behavior.
- Admin/security mutation handlers synchronously insert `cloud_auth_audit_log` success events after authoritative mutation calls, avoiding false success audit records when storage validation/mutation fails. If post-mutation audit insertion fails, handlers return `500` so callers know the operation did not complete cleanly.

### Persisted task checkbox updates

The following task lines are now visibly checked in `openspec/changes/cloud-user-token-management/tasks.md`:

- [x] RED: Add admin authorization tests in `internal/cloud/cloudserver/` proving only managed admin principals can create users, issue/revoke tokens, and create/revoke grants; members receive forbidden responses and no state changes.
- [x] GREEN: Add admin form/API handlers under `internal/cloud/cloudserver/` for human user create/list/enable/disable, token create/list/revoke, and grant create/list/revoke, backed by cloudstore methods.

The broader PR3 audit task lines remain unchecked because they also include admin login, dashboard bootstrap, and legacy recovery audit coverage, which are explicitly out of scope for PR3A.

### TDD Cycle Evidence

| Task | Test File | Layer | Safety Net | RED | GREEN | TRIANGULATE | REFACTOR |
|------|-----------|-------|------------|-----|-------|-------------|----------|
| PR3A admin authorization | `internal/cloud/cloudserver/admin_handlers_test.go` | Handler integration with fakes | ✅ `go test ./internal/cloud/cloudserver ./internal/cloud/auth ./internal/cloud/cloudstore` passed before production edits | ✅ `go test ./internal/cloud/cloudserver -run 'TestAdminHandlers|TestAdminMutations'` failed to compile on missing `WithAdminIdentityStore` / `WithManagedTokenHasher` | ✅ Targeted admin handler tests passed after adding cloudserver options/routes/handlers | ✅ Added legacy-admin forbidden case in addition to managed member forbidden case; covered seven mutation routes and no-state-change assertions | ✅ Kept policy check in `requireManagedAdmin` and store boundary in `AdminIdentityStore` |
| PR3A storage-backed admin APIs | `internal/cloud/cloudserver/admin_handlers_test.go` | Handler integration with fake storage | ✅ Existing cloudserver/auth/cloudstore packages green | ✅ Tests referenced absent admin routes/options before implementation; later metadata-shape refactor RED failed until `token_prefix` was snake_case | ✅ `go test ./internal/cloud/cloudserver -run 'TestAdminHandlers|TestAdminMutations'` passed | ✅ Covered create/list/enable/disable users, create/list/revoke tokens, create/list/revoke grants, show-once raw token, snake_case token metadata, and metadata redaction | ✅ Sanitized token metadata into a dedicated response DTO that omits hash/raw fields |
| PR3A admin mutation audit | `internal/cloud/cloudserver/admin_handlers_test.go` | Handler integration with fake storage/audit | ✅ Existing cloudserver/auth/cloudstore packages green | ✅ Audit tests failed before admin handlers existed | ✅ Targeted admin handler tests passed after synchronous audit insertion | ✅ Covered all PR3A mutation action names and audit-insert failure preventing mutation | ✅ Centralized audit insertion in `recordAdminAudit`; no raw token/hash/bearer metadata is emitted |

### Test Summary

- Total tests written: 3 handler integration tests with subcases.
- Total tests passing: targeted admin handler tests, cloudserver/auth/cloudstore package tests, and full repository test suite passed.
- Layers used: Handler integration (3), Unit (0), E2E (0).
- Approval tests: None — this slice added new API behavior rather than refactoring existing behavior.
- Pure functions created: small response/audit/helper functions in `admin_handlers.go` (`sanitizeToken`, `sanitizeTokenList`, `decodeJSONBody`).

### Verification run

```bash
go test ./internal/cloud/cloudserver ./internal/cloud/auth ./internal/cloud/cloudstore
go test ./internal/cloud/cloudserver -run 'TestAdminHandlers|TestAdminMutations'
go test ./...
git diff --check
```

Results:

- `go test ./internal/cloud/cloudserver ./internal/cloud/auth ./internal/cloud/cloudstore`: PASS.
- `go test ./internal/cloud/cloudserver -run 'TestAdminHandlers|TestAdminMutations'`: PASS.
- `go test ./...`: PASS.
- `git diff --check`: PASS.

### Files changed

- `internal/cloud/cloudserver/admin_handlers.go`
- `internal/cloud/cloudserver/admin_handlers_test.go`
- `internal/cloud/cloudserver/cloudserver.go`
- `openspec/changes/cloud-user-token-management/tasks.md`
- `openspec/changes/cloud-user-token-management/apply-progress.md`

### Changed-line estimate

- Code/test diff before OpenSpec updates: approximately 826 added lines (`admin_handlers.go` 406 lines, `admin_handlers_test.go` 396 lines, `cloudserver.go` +24 lines).
- Including task/progress artifact updates, the slice remains under the parent stop threshold of 900 implementation changed lines and does not include dashboard/bootstrap/CLI expansion.

### Deviations from design

- Admin handlers were implemented as JSON/API form-level `/admin/*` cloudserver endpoints for PR3A reviewability, not dashboard-rendered `/dashboard/admin/*` UX/templates. PR4 owns managed-user dashboard UX.
- Cloudserver exposes explicit wiring options for the admin identity store and token hasher. Runtime CLI/config wiring for a dedicated managed-token pepper is not expanded in PR3A.
- Audit insertion is synchronous and happens after mutation calls to avoid false success audit records. PR3A does not expand storage into multi-operation admin transactions, so a post-mutation audit insertion failure returns `500` after the authoritative mutation has occurred.

### Remaining tasks

Exact unchecked task lines remaining in `tasks.md` after PR3A:

```markdown
- [ ] RED: Add dashboard-session tests proving managed admin login succeeds, member admin access fails, disabled/demoted users lose access on the next protected request, and secure cookie attributes are set correctly.
- [ ] GREEN: Update dashboard auth/session handling in `internal/cloud/cloudserver` and `internal/cloud/dashboard` so signed cookies carry principal claims but every protected request revalidates enabled state and role from storage.
- [ ] RED: Add bootstrap tests for legacy dashboard/admin credential creating the first managed admin, rejecting duplicate first-admin bootstrap, and preventing accidental removal of the last usable managed admin path.
- [ ] GREEN: Implement dashboard bootstrap route/handler and last-admin protections, treating `ENGRAM_CLOUD_ADMIN` as explicit bootstrap/recovery access after managed admins exist.
- [ ] RED: Add audit tests for token create/revoke, user create/enable/disable, grant create/revoke, admin login, dashboard bootstrap, accepted/rejected legacy recovery actions, and redaction of raw tokens.
- [ ] GREEN: Emit synchronous `cloud_auth_audit_log` events for admin/security mutations; fail authoritative admin mutations if audit insertion fails.
- [ ] Verify: targeted admin/dashboard/bootstrap tests and `go test ./...`.
- [ ] Rollback boundary: remove admin/bootstrap routes while retaining storage/auth foundation and legacy auth behavior.
- [ ] RED: Add dashboard rendering/handler tests for `/dashboard/admin/users`, `/dashboard/admin/users/list`, token partials, grant partials, and contributor/managed-user separation.
- [ ] GREEN: Update `internal/cloud/dashboard/dashboard.go` and related templ/templates/assets to show `Managed Users` separately from contributor analytics.
- [ ] GREEN: Add server-rendered forms and HTMX-compatible partials for user create, enable/disable, token create/show-once, token revoke, grant create, and grant revoke.
- [ ] TRIANGULATE: Test non-HTMX form POST/redirect behavior and HTMX partial responses; partials must be meaningful HTML without hidden client-side policy logic.
- [ ] TRIANGULATE: Test empty states explaining deny-by-default project grants and token show-once warnings.
- [ ] REFACTOR: Keep policy checks in server/auth/store layers; dashboard code must render outcomes, not make authorization decisions.
- [ ] Verify: dashboard package tests plus `go test ./...`.
- [ ] Rollback boundary: remove dashboard UI routes/templates without affecting already-tested admin handlers.
- [ ] RED: Add CLI tests in `cmd/engram/` for `engram cloud bootstrap admin --username ...`, duplicate bootstrap refusal, optional token issuance printed once, optional project grants, invalid input, and audit event creation.
- [ ] GREEN: Implement `engram cloud bootstrap admin` in `cmd/engram/cloud.go`, using cloud runtime DB configuration by default and an existing DSN override convention only if already present.
- [ ] TRIANGULATE: Test that raw managed tokens are never persisted, logged, audited, rendered in token metadata lists, or printed except the creation/bootstrap response.
- [ ] GREEN: Update docs discovery targets affected by cloud setup and sync auth, starting with `README.md`, `docs/`, `CONTRIBUTING.md`, and any cloud deployment docs found by `rg "ENGRAM_CLOUD_TOKEN|ENGRAM_CLOUD_ADMIN|ENGRAM_CLOUD_ALLOWED_PROJECTS|cloud bootstrap"`.
- [ ] GREEN: Document managed users/tokens, dedicated token pepper, first-admin dashboard bootstrap, CLI bootstrap, project grants, deny-by-default managed principals, legacy env-token migration, and rollback to legacy sync credentials.
- [ ] RED: Add regression tests that `/sync/*` route methods, paths, request schemas, and response schemas remain unchanged for existing clients.
- [ ] GREEN: Fix any contract drift found by regression tests without changing MVP payloads.
- [ ] REFACTOR: Run `gofmt` on touched Go files and remove any temporary test seams not needed by production behavior.
- [ ] Verify: `go test ./...`, targeted cloud tests (`go test ./internal/cloud/... ./cmd/engram`), and `go test -cover ./...`.
- [ ] Rollback boundary: revert CLI/docs/audit hardening slice while keeping prior reviewed server behavior intact.
- [ ] Managed human users are distinct from contributor analytics.
- [ ] Managed tokens authenticate principals; authorization uses principal role and project grants.
- [ ] Token hashes use a dedicated cloud token pepper, not the dashboard/session signing secret.
- [ ] Raw token values are shown once and never stored or audited.
- [ ] Disabled users, revoked tokens, and revoked grants stop future access immediately.
- [ ] Legacy `ENGRAM_CLOUD_TOKEN`, `ENGRAM_CLOUD_ADMIN`, and `ENGRAM_CLOUD_ALLOWED_PROJECTS` behavior remains compatible during migration.
- [ ] Managed principals are deny-by-default for project sync.
- [ ] Dashboard cookies are `HttpOnly`, `SameSite=Lax` or stronger, and `Secure` under HTTPS/production rules.
- [ ] CLI and dashboard can create the first managed admin safely.
- [ ] Audit events cover all required MVP identity/security actions without secret leakage.
- [ ] Documentation matches real routes, commands, environment variables, and rollback behavior.
```

### Risks / follow-ups

- PR3A intentionally does not wire dashboard sessions, bootstrap, CLI, or dashboard UX. Those remain the next PR3/PR4/PR5 slices.
- Runtime serving still needs dedicated managed-token pepper/config wiring before token creation can be enabled outside explicit `WithManagedTokenHasher` construction.
- The PR3A admin audit implementation avoids false success audit records by auditing after successful mutations, but it does not make audit+mutation a single database transaction; if audit insertion fails after mutation, the API returns `500` after state changed. A later hardening slice can add transactional composite store methods if needed.
