# Feature Specification: MCP OAuth 2.1 Authentication

**Feature Branch**: `mcp`
**Created**: 2026-02-20
**Status**: Draft
**Target**: api
**Input**: User description: "Added MCP server with API key auth, but Claude Desktop requires OAuth 2.0 flow for authentication"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Claude Desktop Discovers and Authenticates with MCP Server (Priority: P1)

A user configures Claude Desktop to connect to the procurement API's MCP server. Claude Desktop follows the MCP OAuth 2.1 discovery flow: it hits the MCP endpoint, receives a 401 with resource metadata URL, discovers the authorization server, opens a browser for user login, obtains an access token, and uses it to access MCP tools.

**Why this priority**: This is the core flow — without OAuth 2.1 discovery and authorization code flow, Claude Desktop cannot connect at all.

**Independent Test**: Can be fully tested by pointing Claude Desktop at the MCP server URL and verifying the full discovery-authenticate-connect cycle completes successfully.

**Acceptance Scenarios**:

1. **Given** an unauthenticated request to `/mcp`, **When** Claude Desktop sends a request without a token, **Then** the server responds with HTTP 401 and a `WWW-Authenticate: Bearer resource_metadata="<url>"` header.
2. **Given** Claude Desktop fetches `/.well-known/oauth-protected-resource`, **When** the metadata is returned, **Then** it contains the `authorization_servers` field pointing to the authorization server.
3. **Given** a user completes the OAuth login in their browser, **When** Claude Desktop exchanges the authorization code (with PKCE) for an access token, **Then** the token endpoint returns a valid Bearer token.
4. **Given** a valid Bearer token, **When** Claude Desktop sends requests to `/mcp` with `Authorization: Bearer <token>`, **Then** the server authenticates and grants access to MCP tools.

---

### User Story 2 - User Logs In via Browser During OAuth Flow (Priority: P1)

When Claude Desktop initiates the OAuth flow, the user's browser opens to a login page. The user authenticates with their existing procurement API credentials (email/password). Upon successful login, the browser redirects back and Claude Desktop receives the authorization code.

**Why this priority**: Equal to P1 because without the authorize endpoint and user login, the OAuth flow cannot complete.

**Independent Test**: Can be tested by visiting the authorization URL directly in a browser, logging in, and verifying the redirect contains a valid authorization code.

**Acceptance Scenarios**:

1. **Given** a user is redirected to the authorization endpoint, **When** they enter valid credentials, **Then** they are redirected to the callback URL with an authorization code.
2. **Given** a user enters invalid credentials, **When** they submit the login form, **Then** they see an error message and can retry.
3. **Given** a user denies consent, **When** they click deny, **Then** the redirect includes an error parameter.

---

### User Story 3 - Token Refresh and Expiration (Priority: P2)

When an access token expires during an active MCP session, Claude Desktop uses the refresh token to obtain a new access token without requiring the user to log in again.

**Why this priority**: Important for user experience during long sessions, but not required for initial connection.

**Independent Test**: Can be tested by issuing a short-lived token, waiting for expiry, and verifying the refresh flow returns a new valid token.

**Acceptance Scenarios**:

1. **Given** an expired access token and a valid refresh token, **When** a token refresh request is made, **Then** a new access token is returned.
2. **Given** an expired refresh token, **When** a token refresh request is made, **Then** the server responds with 401 and the user must re-authenticate.

---

### Edge Cases

- What happens when the PKCE code verifier does not match the code challenge? (Must reject with error)
- What happens when the authorization code is used more than once? (Must reject second use)
- What happens when the redirect URI does not match the registered one? (Must reject)
- How does the system handle concurrent OAuth flows from the same user?
- What happens if the MCP session outlives the access token and no refresh token was issued?

## Requirements *(mandatory)*

### Functional Requirements

#### Discovery & Metadata
- **FR-001**: System MUST serve a Protected Resource Metadata document at `/.well-known/oauth-protected-resource` containing the `authorization_servers` field and supported scopes.
- **FR-002**: System MUST return HTTP 401 with `WWW-Authenticate: Bearer resource_metadata="<url>"` header for unauthenticated requests to the MCP endpoint.
- **FR-003**: System MUST serve OAuth Authorization Server Metadata at `/.well-known/oauth-authorization-server` describing available endpoints and supported capabilities.

#### Authorization Endpoints
- **FR-004**: System MUST expose an authorization endpoint (`/authorize`) that initiates the OAuth 2.1 Authorization Code flow with PKCE.
- **FR-005**: System MUST expose a token endpoint (`/token`) that exchanges authorization codes for access tokens and handles token refresh.
- **FR-006**: System MUST enforce PKCE with `S256` code challenge method on all authorization requests.
- **FR-007**: System MUST present a login page when the user is redirected to the authorization endpoint, using existing procurement API user credentials.

#### Token Validation
- **FR-008**: System MUST validate Bearer tokens on every request to the MCP endpoint, checking token validity and expiration.
- **FR-009**: System MUST issue access tokens with a configurable time-to-live and support refresh tokens for session continuity.
- **FR-010**: System MUST return HTTP 403 when a valid token lacks sufficient scopes for the requested operation.

#### Client Registration
- **FR-011**: System MUST support Dynamic Client Registration (RFC 7591) to allow MCP clients like Claude Desktop to self-register. [NEEDS CLARIFICATION: Should the server support Dynamic Client Registration for any client, or should Claude Desktop be pre-registered as a known client?]

#### Security
- **FR-012**: System MUST reject authorization codes that have already been used.
- **FR-013**: System MUST log all OAuth authentication events (authorization requests, token issuance, token refresh, failures) for auditing.

### Key Entities

- **OAuth Client**: An MCP client (e.g., Claude Desktop) that registers with the authorization server. Attributes: client ID, redirect URIs, registration date.
- **Authorization Code**: A short-lived code issued after user login, exchanged for tokens. Attributes: code value, client ID, user, PKCE challenge, expiration, used status.
- **Access Token**: A Bearer token granting access to MCP resources. Attributes: token value (hashed), associated user, client ID, scopes, expiration.
- **Refresh Token**: A long-lived token used to obtain new access tokens. Attributes: token value (hashed), associated user, client ID, expiration.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Claude Desktop can discover, authenticate, and connect to the MCP server through the full OAuth 2.1 flow on the first attempt.
- **SC-002**: Users complete the browser-based login during OAuth flow in under 30 seconds.
- **SC-003**: Token exchange (authorization code to access token) completes in under 2 seconds.
- **SC-004**: Expired tokens are seamlessly refreshed without user intervention during active sessions.
- **SC-005**: 100% of unauthorized requests are correctly rejected with appropriate HTTP status codes (401/403).

## Assumptions

- Claude Desktop implements the MCP OAuth 2.1 specification including PKCE with S256 and resource indicators (RFC 8707).
- The existing user model and credentials in the procurement API will be used for the authorization login page.
- The authorization server will be built into the procurement API (co-located), not a separate external service like Auth0 or Keycloak.
- The existing JWT infrastructure (Lexik JWT) can be leveraged for issuing OAuth access tokens.
- The Symfony security system can accommodate a new OAuth Bearer token authenticator alongside the existing JWT and API key authenticators.
