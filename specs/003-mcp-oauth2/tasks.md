# Tasks: MCP OAuth2 User Authentication

**Input**: Design documents from `/specs/003-mcp-oauth2/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/oauth-endpoints.md

**Tests**: Not explicitly requested — test tasks are omitted.

**Organization**: Tasks are grouped by user story. US1 (OAuth2 connect flow) is the MVP. US2 (token refresh) builds on US1.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2)
- Include exact file paths in descriptions

## Path Conventions

- **Symfony API**: `src/`, `config/`, `migrations/` at repository root (`procurement-api`)
- **React Frontend**: `src/` at repository root (`procurement-app` — separate repo at `/home/aelfannir/dev/procurement-app/`)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Install dependencies, configure the OAuth2 bundle, generate database tables

- [x] T001 Install `league/oauth2-server-bundle`, `nyholm/psr7`, and `symfony/psr-http-message-bridge` via `docker compose exec php composer require league/oauth2-server-bundle nyholm/psr7 symfony/psr-http-message-bridge`
- [x] T002 Create bundle configuration in `config/packages/league_oauth2_server.yaml` — auth code + refresh token grants, reuse existing RSA keys (`%env(resolve:JWT_SECRET_KEY)%`, `%env(resolve:JWT_PUBLIC_KEY)%`), access token TTL 1h, refresh token TTL 30d, auth code TTL 10m, single scope `mcp:tools`
- [x] T003 Add env vars to `.env`: `OAUTH2_ENCRYPTION_KEY` (generate via `vendor/bin/generate-defuse-key`), `FRONTEND_URL=http://localhost:3000`, `OAUTH2_ISSUER=http://localhost:84`
- [x] T004 Create `OAuth2AuthorizationContext` entity in `src/Entity/OAuth2AuthorizationContext.php` with fields: id (PK, SEQUENCE), authCodeIdentifier (string 80, indexed), user (ManyToOne → User, NOT NULL), workspace (string 20, nullable), countryCode (string 5, nullable), clinicId (int, nullable), createdAt (datetime, NOT NULL)
- [x] T005 Create repository in `src/Repository/OAuth2AuthorizationContextRepository.php` with `findByAuthCodeIdentifier(string $identifier)` method
- [x] T006 Add Doctrine mapping for bundle entities in `config/packages/doctrine.yaml` under `orm.mappings` — prefix `League\Bundle\OAuth2ServerBundle\Entity`, dir `%kernel.project_dir%/vendor/league/oauth2-server-bundle/src/Entity`
- [x] T007 Generate and run database migration via `docker compose exec php bin/console doctrine:migrations:diff` then `doctrine:migrations:migrate` — creates bundle tables (oauth2_client, oauth2_access_token, oauth2_refresh_token, oauth2_authorization_code) + oauth2_authorization_context table
- [x] T008 Create OAuth route configuration in `config/routes/oauth.yaml` — import `src/Controller/OAuth/` with attribute type, no prefix (so `.well-known/*` and `/oauth/*` are at root, not under `/custom`)
- [x] T009 Create bundle token endpoint route in `config/routes/league_oauth2_server.yaml` — import `@LeagueOAuth2ServerBundle/Resources/config/routes.xml` with prefix `/oauth`
- [x] T010 Create service wiring in `config/services/oauth.yaml` — bind `$oauth2Issuer` to `%env(OAUTH2_ISSUER)%`, `$frontendUrl` to `%env(FRONTEND_URL)%` for OAuth controllers and listeners. Also add `imports: [{ resource: 'services/oauth.yaml' }]` to `config/services.yaml` so Symfony discovers the new file

**Checkpoint**: Bundle installed, database tables created, routes configured. No endpoints functional yet.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Metadata discovery endpoints and security infrastructure that ALL user stories depend on

**CRITICAL**: No user story work can begin until this phase is complete

- [x] T011 Create `MetadataController` in `src/Controller/OAuth/MetadataController.php` with two actions: (1) `GET /.well-known/oauth-protected-resource` returns Protected Resource Metadata (RFC 9728) JSON: resource, authorization_servers, scopes_supported; (2) `GET /.well-known/oauth-authorization-server` returns Authorization Server Metadata (RFC 8414) JSON: issuer, authorization_endpoint, token_endpoint, scopes_supported, response_types_supported, grant_types_supported, token_endpoint_auth_methods_supported, code_challenge_methods_supported, client_id_metadata_document_supported — both per contracts/oauth-endpoints.md
- [x] T012 [P] Create `OAuth2Authenticator` in `src/Security/OAuth2Authenticator.php` — extends AbstractAuthenticator, validates OAuth2 Bearer tokens using league's `ResourceServer`, converts Symfony request to PSR-7 via `PsrHttpFactory`, resolves user by email from `UserRepository`, stores oauth2_user_id and oauth2_scopes on request attributes. `supports()` must detect JWT format (contains dots) to differentiate from opaque API key tokens. MUST validate token audience (`aud` claim) matches `OAUTH2_ISSUER` env var (RFC 8707) — reject tokens issued for other resources. MUST NOT pass through received tokens to downstream services (FR-012)
- [x] T013 [P] Modify `ApiKeyAuthenticator` in `src/Security/ApiKeyAuthenticator.php` — update `supports()` to return false when token looks like a JWT (contains dots), so it only handles opaque API key tokens. This ensures coexistence with OAuth2Authenticator
- [x] T014 [P] Create `OAuth2ChallengeListener` in `src/EventListener/OAuth2ChallengeListener.php` — listens to `kernel.exception` events, on 401 responses for `/mcp` and `/graphql` paths adds `WWW-Authenticate: Bearer resource_metadata="{issuer}/.well-known/oauth-protected-resource", scope="mcp:tools"` header
- [x] T015 Update `config/packages/security.yaml` — add `well_known` firewall (pattern `^/.well-known/`, security false), add `oauth_public` firewall (pattern `^/oauth/(authorize|token|deny)`, security false), add `oauth_approve` firewall (pattern `^/oauth/approve`, jwt auth), update `mcp` firewall to use both `OAuth2Authenticator` + `ApiKeyAuthenticator` with `entry_point: App\Security\OAuth2Authenticator`, update `graphql` firewall same way, add access_control rules for `^/.well-known/` (PUBLIC_ACCESS), `^/oauth/authorize` (PUBLIC_ACCESS), `^/oauth/token` (PUBLIC_ACCESS), `^/oauth/approve` (IS_AUTHENTICATED_FULLY)

**Checkpoint**: Metadata endpoints respond correctly, OAuth2 authenticator validates tokens on MCP/GraphQL firewalls, API key auth still works for `/custom/invoices/liasse`, WWW-Authenticate headers appear on 401 responses.

---

## Phase 3: User Story 1 — User connects AI agent via OAuth2 (Priority: P1) MVP

**Goal**: Complete OAuth2 authorization code flow — from discovery through consent to authenticated MCP requests with user context.

**Independent Test**: Connect Claude Desktop to `http://localhost:84/mcp`. Complete the OAuth2 flow (browser opens, user logs in, consents, selects workspace/country). Use `graphql_query` tool to verify results are filtered by the authenticated user's clinic/section permissions.

### Authorization Flow (Symfony API)

- [x] T016 Create `AuthorizeController` in `src/Controller/OAuth/AuthorizeController.php` — `GET /oauth/authorize` validates required OAuth params (client_id, redirect_uri, response_type=code, code_challenge, code_challenge_method=S256, state), validates scope defaults to `mcp:tools`, then redirects to `{FRONTEND_URL}/oauth/consent?{all_params}`. On validation errors: return error to redirect_uri per RFC 6749 section 4.1.2.1, or 400 JSON if redirect_uri is invalid
- [x] T017 Create `ApproveController` in `src/Controller/OAuth/ApproveController.php` — `POST /oauth/approve` (JWT-protected) receives OAuth params + context (workspace, country_code, clinic_id) from React frontend, uses league's `AuthorizationServer` to complete the auth code grant (convert Symfony request to PSR-7), creates `OAuth2AuthorizationContext` entity linking auth_code_identifier to the selected context and authenticated user, returns `{ "redirect_url": "{redirect_uri}?code=xxx&state=yyy" }`
- [x] T018 Add deny action to `src/Controller/OAuth/ApproveController.php` — `POST /oauth/deny` (JWT-protected) receives client_id, redirect_uri, state and returns `{ "redirect_url": "{redirect_uri}?error=access_denied&error_description=...&state=yyy" }`

### Context in Access Tokens

- [x] T019 Create `CustomAccessTokenRepository` in `src/OAuth2/CustomAccessTokenRepository.php` — decorates `league.oauth2_server.repository.access_token`, in `getNewToken()` looks up `OAuth2AuthorizationContext` by the user identifier and most recent auth code, sets private claims (workspace, country_code, clinic_id) on the token via `setPrivateClaims()`
- [x] T020 Register decorator in `config/services/oauth.yaml` — `App\OAuth2\CustomAccessTokenRepository` decorates `league.oauth2_server.repository.access_token` with `$inner: '@.inner'`
- [x] T021 Create `OAuth2ContextService` in `src/Service/OAuth2ContextService.php` — reads request attributes (oauth2_workspace, oauth2_country_code, oauth2_clinic_id) set by the authenticator, provides `getWorkspace()`, `getCountryCode()`, `getClinicId()` methods with header override precedence (Tenant-Module, Tenant-Clinic, Accept-Language headers take priority over token claims)
- [x] T022 Update `OAuth2Authenticator` in `src/Security/OAuth2Authenticator.php` to decode JWT private claims (workspace, country_code, clinic_id) from the validated token and set them as request attributes (`oauth2_workspace`, `oauth2_country_code`, `oauth2_clinic_id`) so `OAuth2ContextService` and `LocaleService` can use them
- [x] T023 Update `LocaleService` in `src/Service/LocaleService.php` — inject `OAuth2ContextService`, modify `getWorkspace()` to fall back to `OAuth2ContextService->getWorkspace()` when `Tenant-Module` header is absent, modify `getCountryCode()` to fall back to `OAuth2ContextService->getCountryCode()` when `Accept-Language` header is absent, modify `getUserClinicIds()` to check `OAuth2ContextService->getClinicId()` before falling back to user's clinics

### Client ID Metadata Document Validation (SHOULD)

- [x] T024 Create `ClientMetadataValidator` in `src/Service/ClientMetadataValidator.php` — when `client_id` starts with `https://`, fetches the URL via `HttpClientInterface`, validates JSON response (`client_id` matches URL, `redirect_uris` contains requested redirect_uri, required fields present), auto-registers the client in the database if not already present. Called from `AuthorizeController` during `/oauth/authorize`

### React Frontend Consent Page

- [x] T025 [P] [US1] Create consent page component in `/home/aelfannir/dev/procurement-app/src/pages/oauth/consent.page.tsx` — parses URL search params (client_id, scope, state, redirect_uri, etc.), checks `useAuth().token`: if not logged in, redirects to `/auth/login?returnUrl=/oauth/consent?{params}`. If logged in, renders consent form using `Login2Layout` with: client name display, scope description, workspace selector (button grid like SwitchWorkspacePage using `APP_ENUM_OPTIONS`), country selector (from `useLocale().countries`), optional clinic selector (lazy fetch filtered by workspace via react-query), Authorize/Deny buttons
- [x] T026 [US1] Add Authorize/Deny handlers to consent page — Authorize: `POST /oauth/approve` with JWT (auto-attached by axios interceptor) + all OAuth params + context selections → `window.location.href` to returned redirect_url. Deny: `POST /oauth/deny` → `window.location.href` to returned redirect_url
- [x] T027 [US1] Register `/oauth/consent` route in `/home/aelfannir/dev/procurement-app/src/config/routing/AppRoutes.tsx` as a top-level route (alongside `/error`, BEFORE the `/*` catch-all) so it's accessible regardless of auth state
- [x] T028 [US1] Update login page in `/home/aelfannir/dev/procurement-app/src/pages/auth/login/login.page.tsx` — after successful login (`onSuccess` callback), check for `returnUrl` search param and `navigate(returnUrl)` instead of default routing
- [x] T029 [P] [US1] Create i18n translation files: `/home/aelfannir/dev/procurement-app/src/i18n/locales/en/oauth-consent.json`, `fr/oauth-consent.json`, `ar/oauth-consent.json` — keys: title, description, scopes.label, scopes.mcp:tools, workspace.label, country.label, clinic.label, authorize, deny, error.invalid_request, error.authorize_failed
- [x] T030 [US1] Register oauth-consent namespace in locale index files: `/home/aelfannir/dev/procurement-app/src/i18n/locales/en/index.ts`, `fr/index.ts`, `ar/index.ts`

### Create Test Client

- [x] T031 [US1] Create a pre-registered test OAuth2 client via `docker compose exec php bin/console league:oauth2-server:create-client "mcp-test-client" --grant-type=authorization_code --grant-type=refresh_token --scope=mcp:tools --redirect-uri=http://127.0.0.1/callback`

**Checkpoint**: Full OAuth2 flow works end-to-end. MCP client discovers auth server, user consents with context selection, token carries user identity + context, MCP queries return user-filtered data.

---

## Phase 4: User Story 2 — Token refresh without user intervention (Priority: P2)

**Goal**: Refresh tokens work correctly — AI agents can maintain sessions without re-login. Public client refresh tokens are rotated.

**Independent Test**: After completing the OAuth2 flow, wait for the access token to expire (or manually set short TTL). Verify the AI agent automatically refreshes via the token endpoint and continues making MCP requests without user interaction.

- [x] T032 [US2] Verify refresh token grant is enabled in `config/packages/league_oauth2_server.yaml` (`enable_refresh_token_grant: true`) and that the bundle's token endpoint at `/oauth/token` accepts `grant_type=refresh_token` requests with proper `client_id` and `resource` parameter
- [x] T033 [US2] Verify that `CustomAccessTokenRepository` in `src/OAuth2/CustomAccessTokenRepository.php` correctly injects context claims into refreshed access tokens — when a refresh token is used, the new access token should carry the same workspace/country_code/clinic_id claims from the original authorization context
- [x] T034 [US2] Verify refresh token rotation for public clients — confirm the bundle configuration rotates refresh tokens on use (old token invalidated, new token issued). Test by using a refresh token twice: the second use should fail

**Checkpoint**: Refresh tokens work. Public client tokens rotate. Context claims persist across token refreshes.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Edge cases, security hardening, compatibility verification

- [x] T035 Verify existing `api_key` firewall on `/custom/invoices/liasse` still works unchanged — the `ApiKeyAuthenticator` should only activate for opaque tokens (not JWTs)
- [x] T036 Verify disabled user accounts are rejected during token validation — `OAuth2Authenticator` should check user active status after resolving from UserRepository. Also verify FR-012: tokens received on MCP/GraphQL endpoints are never passed through to downstream services
- [x] T037 Add CORS headers for OAuth endpoints if needed — check `config/packages/nelmio_cors.yaml` covers `/.well-known/*` and `/oauth/*` paths (the existing `^/` catch-all pattern should already cover this)
- [x] T038 Run the quickstart.md validation — follow all steps in `specs/003-mcp-oauth2/quickstart.md` from scratch and verify the complete flow works with Claude Desktop

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately
- **Foundational (Phase 2)**: Depends on Setup (Phase 1) completion — BLOCKS all user stories
- **User Story 1 (Phase 3)**: Depends on Foundational (Phase 2) — this is the MVP
- **User Story 2 (Phase 4)**: Depends on User Story 1 (Phase 3) — refresh tokens require the auth code flow to work first
- **Polish (Phase 5)**: Depends on both user stories being complete

### Within Phase 1 (Sequential)

T001 → T002 → T003 → T004, T005 (parallel) → T006 → T007 → T008, T009, T010 (parallel)

### Within Phase 2 (Parallel where marked)

T011 (MetadataController — both actions) → T012, T013, T014 (parallel — different files) → T015 (depends on all authenticators/listeners existing)

### Within Phase 3 (US1)

API side: T016 → T017 → T018 → T019 → T020 → T021 → T022 → T023 → T024
React side: T025, T029 (parallel — different files) → T026 (depends on T025) → T027 → T028 → T030
Test client: T031 (after API side complete)

**API and React sides can be developed in parallel** — they only integrate at the HTTP boundary (`POST /oauth/approve`, `POST /oauth/deny`).

### Parallel Opportunities

```bash
# Phase 2 — after T011 (MetadataController), authenticator + listener in parallel:
Task: T012 "OAuth2Authenticator"
Task: T013 "ApiKeyAuthenticator coexistence"
Task: T014 "OAuth2ChallengeListener"

# Phase 3 — React consent page in parallel with API authorize flow:
Task: T025 "React consent page component"
Task: T029 "React i18n translations"
# Then sequentially: T026 (handlers, same file as T025) → T027 → T028 → T030
# While API works on T016-T024
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (bundle, config, migration)
2. Complete Phase 2: Foundational (metadata, authenticator, firewall)
3. Complete Phase 3: User Story 1 (authorize flow + consent page + context)
4. **STOP and VALIDATE**: Test full OAuth2 flow with Claude Desktop
5. Deploy/demo if ready

### Incremental Delivery

1. Setup + Foundational → Infrastructure ready
2. Add User Story 1 → Full OAuth2 flow works → Deploy (MVP!)
3. Add User Story 2 → Token refresh works → Deploy
4. Polish → Edge cases and security hardening → Final release

---

## Notes

- [P] tasks = different files, no dependencies
- [US1/US2] labels map tasks to specific user stories
- React frontend work (`procurement-app`) is in a separate repository — file paths use absolute paths
- The `league/oauth2-server-bundle` provides the token endpoint controller — no custom code needed for `/oauth/token`
- The `mcp` and `graphql` firewalls keep `ApiKeyAuthenticator` as fallback during transition
- Commit after each task or logical group
- Stop at any checkpoint to validate the story independently
