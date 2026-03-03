# Feature Specification: MCP OAuth2 User Authentication

**Feature Branch**: `003-mcp-oauth2`
**Created**: 2026-02-25
**Status**: Draft
**Target**: api
**Input**: User description: "Add OAuth2 authentication for the MCP endpoint so each user authenticates individually via Claude Desktop's OAuth2 flow, replacing the shared API key approach"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - User connects AI agent to procurement system via OAuth2 (Priority: P1)

A user opens their AI agent (Claude Desktop, ChatGPT, VS Code, etc.) and adds the procurement MCP server URL. The agent discovers the OAuth2 authorization server via the MCP protocol's metadata discovery flow:

1. Agent sends initial request to MCP endpoint → receives `401` with `WWW-Authenticate` header pointing to Protected Resource Metadata
2. Agent fetches Protected Resource Metadata (RFC 9728) → discovers authorization server URL and supported scopes
3. Agent fetches Authorization Server Metadata (RFC 8414) → discovers authorize, token endpoints and capabilities (including `code_challenge_methods_supported`)
4. Agent resolves client registration (Client ID Metadata Document, pre-registered credentials, or DCR fallback)
5. Agent opens browser to authorization endpoint with PKCE challenge + resource parameter → user logs in with email/password and consents
6. Agent exchanges authorization code for access token (with PKCE verifier + resource parameter)
7. Agent uses access token (with audience bound to this MCP server) for all subsequent MCP requests

The system identifies the real user and applies their clinic restrictions, section category restrictions, and workspace access — just like the web frontend does.

**Why this priority**: Without user identification, all MCP queries return empty results because data access filters cannot determine which clinics and sections the user has access to. OAuth2 is the standard protocol that AI agents expect from remote MCP servers (per the MCP authorization specification, protocol revision 2025-11-25).

**Independent Test**: Can be tested by completing the OAuth2 flow in Claude Desktop, then using the graphql_query tool and verifying that results are filtered by the authenticated user's permissions.

**Acceptance Scenarios**:

1. **Given** a user with valid credentials, **When** they connect an AI agent to the MCP server, **Then** the agent discovers the auth server via metadata, the user is redirected to a login/consent page, and can authorize the agent.
2. **Given** a user who has completed the OAuth2 flow, **When** the AI agent makes MCP requests with the access token, **Then** the system authenticates as that user with their full permissions.
3. **Given** a user with `clinicRestrictionEnabled=true` and assigned clinics, **When** the AI agent queries purchase orders via MCP, **Then** only purchase orders from those clinics are returned.
4. **Given** a user with `sectionRestrictionEnabled=true`, **When** the AI agent queries products, **Then** only products in the user's assigned sections are returned.
5. **Given** an expired or revoked access token, **When** the AI agent makes an MCP request, **Then** the server responds with `401` and proper `WWW-Authenticate` challenge including `resource_metadata`.

---

### User Story 2 - Token refresh without user intervention (Priority: P2)

After initial authorization, the AI agent uses a refresh token to obtain new access tokens without requiring the user to log in again. Sessions persist across agent restarts as long as the refresh token is valid.

**Why this priority**: Without refresh tokens, users would need to re-authenticate frequently, disrupting the AI agent experience.

**Independent Test**: Can be tested by waiting for an access token to expire and verifying the agent automatically obtains a new one via the token endpoint.

**Acceptance Scenarios**:

1. **Given** a valid refresh token, **When** the access token expires, **Then** the AI agent can obtain a new access token without user interaction.
2. **Given** a revoked refresh token, **When** the agent attempts to refresh, **Then** the refresh fails and the user must re-authorize.
3. **Given** a public client (no client secret), **When** a refresh token is used, **Then** the server rotates the refresh token (issues a new one and invalidates the old).

---

### Edge Cases

- What happens when a user has no clinics assigned and `clinicRestrictionEnabled=true`? Queries return empty results (consistent with web frontend behavior).
- What happens when the user's account is disabled after OAuth2 authorization? Token validation should check the user's active status and reject disabled accounts.
- What happens when multiple AI agents connect for the same user? Each agent gets its own token pair — this is valid and expected.
- What happens to existing API key usage on `/custom/invoices/liasse`? The existing `api_key` firewall continues to work unchanged.
- What happens to the existing MCP API key firewall? It is replaced by the OAuth2 firewall for MCP endpoints.
- What if a client sends an invalid or missing PKCE code_verifier? The token request is rejected per OAuth 2.1.
- What if a token's audience doesn't match this MCP server? The request is rejected — tokens must be audience-bound (RFC 8707).
- What if a client omits the `resource` parameter? The authorization/token request proceeds but the token may not be audience-bound; the server still validates audience on incoming requests.

## Requirements *(mandatory)*

### Functional Requirements

**MCP Protocol Compliance (metadata & discovery)**:
- **FR-001**: System MUST serve Protected Resource Metadata (RFC 9728) describing the MCP resource server, its authorization server, and supported scopes. Discovery via either `WWW-Authenticate` header or well-known URI.
- **FR-002**: System MUST serve Authorization Server Metadata (RFC 8414) at `/.well-known/oauth-authorization-server` listing the authorize endpoint, token endpoint, supported grant types, and `code_challenge_methods_supported`.
- **FR-003**: On unauthenticated MCP requests, the server MUST respond with `401` and a `WWW-Authenticate: Bearer` header including `resource_metadata` pointing to the PRM document and `scope` indicating required scopes.

**OAuth2 Authorization Server**:
- **FR-004**: System MUST implement the OAuth 2.1 authorization code flow with mandatory PKCE (S256 code challenge method).
- **FR-005**: System MUST provide an authorization endpoint that redirects to the React frontend consent page. If the user is already logged in, they see only the consent prompt. If not, they go through the existing login flow first.
- **FR-006**: System MUST provide a token endpoint that exchanges authorization codes for access/refresh tokens, supporting the `resource` parameter (RFC 8707) for audience binding.

**Frontend Consent Page**:
- **FR-014**: The React frontend MUST include an `/oauth/consent` page that displays the requesting AI agent's name, requested scopes, and an "Authorize" / "Deny" button.
- **FR-015**: The consent page MUST call the Symfony API (with the user's existing JWT) to generate an authorization code and redirect the browser back to the MCP client's redirect URI with the code.
- **FR-016**: If the user is not logged in, the consent page MUST redirect to the existing login flow and return to the consent page after authentication.

**User Context (workspace, country, clinic)**:
- **FR-017**: The consent page MUST allow the user to select default context: workspace (Project, Exploitation, or Administration), country/language, and optionally a clinic.
- **FR-018**: The selected defaults MUST be stored with the OAuth2 authorization (as custom token claims or in a server-side session linked to the token).
- **FR-019**: The MCP/GraphQL firewall MUST use these defaults when the AI agent does not send context headers.
- **FR-020**: The AI agent MAY override defaults by sending `Tenant-Module`, `Tenant-Clinic`, and `Accept-Language` headers on individual requests — same as the React frontend does. Headers take precedence over stored defaults.
- **FR-021**: The existing `get_app_context` MCP tool already exposes available workspaces, clinics, and countries, enabling the AI agent to know which values are valid for header overrides.
- **FR-007**: System SHOULD support Client ID Metadata Documents (draft-ietf-oauth-client-id-metadata-document-00) as the primary client registration mechanism, and MAY support Dynamic Client Registration (RFC 7591) as a fallback.
- **FR-008**: System MUST support refresh tokens so AI agents can maintain sessions without repeated user login. For public clients, refresh tokens MUST be rotated on each use.
- **FR-009**: Access tokens MUST be short-lived and carry the user's identity so that data access filters (clinic, section, workspace restrictions) apply correctly.

**Resource Server (token validation)**:
- **FR-010**: The MCP firewall MUST validate OAuth2 access tokens, verify the token audience matches this server (RFC 8707), and resolve the authenticated user with their full roles and permissions.
- **FR-011**: The GraphQL firewall MUST also accept OAuth2 tokens (since the graphql_query MCP tool calls GraphQL internally).
- **FR-012**: The server MUST NOT accept tokens issued for other resources and MUST NOT pass through tokens to downstream services.

**Compatibility**:
- **FR-013**: The existing `api_key` firewall (for `/custom/invoices/liasse`) MUST continue to work unchanged.

### Key Entities

- **OAuth2 Client**: Represents an AI agent application (Claude Desktop, ChatGPT, VS Code, etc.). Registered via Client ID Metadata Document, DCR, or pre-registered. Has client ID, optional client secret, allowed redirect URIs, and granted scopes.
- **Access Token**: Short-lived token linked to a user and client. Audience-bound to the MCP server URI. Carries user identity for data access filtering.
- **Refresh Token**: Long-lived token used to obtain new access tokens without re-authorization. Rotated on use for public clients.
- **Authorization Code**: Temporary code exchanged for tokens during the authorization code flow. Protected by PKCE.
- **User**: Existing entity, no changes. Already has clinics, sectionCategories, environments, role, and restriction flags.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: AI agent queries via MCP return the same data as the authenticated user would see in the web frontend (same clinic/section filtering).
- **SC-002**: 100% of MCP requests are authenticated with a real user identity via OAuth2 — no anonymous access possible.
- **SC-003**: Users can connect their AI agent in under 1 minute via the OAuth2 authorization flow.
- **SC-004**: Access tokens expire and refresh automatically without user intervention.
- **SC-005**: Existing API key usage on non-MCP endpoints remains unaffected.
- **SC-006**: MCP-compliant clients (Claude Desktop, ChatGPT, VS Code) can connect using the standard MCP authorization discovery flow.

## Assumptions

- The OAuth2 authorization server will be built into the Symfony application using `league/oauth2-server-bundle`, not an external service like Keycloak.
- The authorization server reuses the same User entity and database as the existing JWT-based authentication.
- The existing RSA key pair (`config/jwt/private.pem`, `config/jwt/public.pem`) will be reused for OAuth2 token signing (different audience claims prevent token confusion).
- The consent flow delegates to the existing React frontend app. Symfony `/oauth/authorize` redirects to the React app's `/oauth/consent` page. If the user is already logged in (JWT in localStorage), they only see an "Authorize" button. If not, they go through the normal React login flow first. The React app calls a Symfony API endpoint to generate the authorization code (sending the JWT in the Authorization header), then redirects the browser back to the MCP client with the code. This avoids re-login for already-authenticated users.
- A single scope (`mcp:tools`) grants full read access to all procurement data the user can see. Granular per-tool scopes are not needed initially.
- PKCE is mandatory per OAuth 2.1 — the server does not accept authorization code flow without PKCE. S256 is the required code challenge method.
- Client ID Metadata Documents (SHOULD) is the primary client registration path. DCR (MAY) is optional fallback. Pre-registration is always supported.
- The `resource` parameter (RFC 8707) is supported for audience-bound tokens.
- The `Tenant-Module` (workspace) and `Tenant-Clinic` headers will be sent by the AI agent in MCP requests, or defaults will apply per existing LocaleService logic.
- HTTPS is enforced in production; HTTP is acceptable for localhost development only.
- All authorization server endpoints are served over HTTPS in production. Redirect URIs must be localhost or HTTPS.
