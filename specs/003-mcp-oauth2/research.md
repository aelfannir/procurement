# Research: MCP OAuth2 Authentication

## MCP Authorization Specification (Protocol Revision 2025-11-25)

Source: https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization

### MUST Requirements (on MCP server / authorization server)

1. **Protected Resource Metadata (RFC 9728)**: MCP servers MUST implement this to indicate authorization server locations
2. **Authorization Server Metadata**: MUST provide either RFC 8414 or OpenID Connect Discovery 1.0
3. **`code_challenge_methods_supported`**: MUST be included in auth server metadata for MCP compatibility
4. **PKCE**: MUST be enforced, S256 challenge method MUST be used when client is capable
5. **Token audience validation**: MCP servers MUST validate that tokens were issued specifically for them (RFC 8707)
6. **Token passthrough forbidden**: MCP servers MUST NOT pass through received tokens to downstream services
7. **HTTPS**: All auth server endpoints MUST be served over HTTPS. Redirect URIs MUST be localhost or HTTPS.
8. **401 response**: MUST include `WWW-Authenticate` with `Bearer` scheme
9. **Token in Authorization header**: MUST use `Authorization: Bearer <token>`, MUST NOT use URI query string
10. **Authorization in every HTTP request**: Even within the same logical session

### SHOULD Requirements

1. **Client ID Metadata Documents** (draft-ietf-oauth-client-id-metadata-document-00): Auth servers and MCP clients SHOULD support this — it's the preferred client registration mechanism
2. **`scope` in WWW-Authenticate**: Servers SHOULD include scope guidance in 401 responses
3. **Short-lived access tokens**: Auth servers SHOULD issue short-lived tokens
4. **Refresh token rotation**: For public clients, MUST rotate refresh tokens

### MAY Requirements

1. **Dynamic Client Registration (RFC 7591)**: MAY support — it's a fallback, not required
2. **Pre-registration**: Clients SHOULD support static client credentials as an option

### Client Registration Priority Order (per spec)

1. Pre-registered client info (if available for this server)
2. Client ID Metadata Documents (if auth server advertises `client_id_metadata_document_supported`)
3. Dynamic Client Registration (fallback, if `registration_endpoint` in auth server metadata)
4. Prompt user to enter client info manually

### Discovery Flow

```
Client → MCP Server: request without token
MCP Server → Client: 401 + WWW-Authenticate: Bearer resource_metadata="..."

Client → MCP Server: GET resource_metadata URI (or fallback to well-known URIs)
MCP Server → Client: { resource, authorization_servers, scopes_supported }

Client → Auth Server: GET /.well-known/oauth-authorization-server (or OIDC discovery)
Auth Server → Client: { issuer, authorization_endpoint, token_endpoint, ... }

Client: resolve registration (CIMD / DCR / pre-registered)

Client → Browser → Auth Server: /authorize + code_challenge + resource
User: login + consent
Auth Server → Client: authorization code via redirect

Client → Auth Server: /token + code_verifier + resource
Auth Server → Client: { access_token, refresh_token, token_type, expires_in }

Client → MCP Server: Authorization: Bearer <access_token>
MCP Server: validate token, check audience, resolve user
```

### Resource Parameter (RFC 8707)

- Clients MUST include `resource` parameter in both authorization and token requests
- Value is the canonical URI of the MCP server (e.g., `https://api.example.com/mcp`)
- Tokens are audience-bound to that resource
- Servers MUST validate the audience claim

### Scope Selection Strategy

1. Use `scope` from initial `WWW-Authenticate` header (if provided)
2. Fall back to all scopes from `scopes_supported` in PRM document
3. Omit scope if `scopes_supported` is undefined

### Error Handling

| Status | Usage |
|--------|-------|
| 401 | Authorization required or token invalid |
| 403 | Invalid scopes or insufficient permissions (`insufficient_scope` error) |
| 400 | Malformed authorization request |

### Step-Up Authorization

When a client has a token but needs additional scopes:
- Server responds 403 with `WWW-Authenticate: Bearer error="insufficient_scope", scope="needed_scopes"`
- Client initiates re-authorization with increased scope set
- Client retries original request (with retry limit)

## Current Auth Architecture

### Firewalls (security.yaml)
- `login` (`^/custom/auth/login`) — JSON login → JWT
- `token_refresh` (`^/custom/auth/refresh`) — JWT refresh
- `api_key` (`^/custom/invoices/liasse`) — ApiKeyAuthenticator (standalone)
- `mcp` (`^/mcp`) — ApiKeyAuthenticator (to be replaced with OAuth2)
- `graphql` (`^/graphql`) — ApiKeyAuthenticator (to be replaced with OAuth2)
- `api` (`^`) — JWT fallback

### ApiKeyAuthenticator
- Reads `Authorization: Bearer <token>` header
- Looks up ApiKey entity by token
- Creates anonymous UserInterface (empty roles, no user identity)
- Problem: LocaleService gets empty clinic/section IDs → `WHERE 1 = 0`

### LocaleService
- `getUserClinicIds()`: checks Tenant-Clinic header, user's clinics, clinicRestrictionEnabled
- `getUserSectionCategoryIds()`: checks query params, user's sections, sectionRestrictionEnabled
- Works correctly when given a real User entity — no changes needed

### Existing OAuth2 packages
- `league/oauth2-client` 2.9.0 — OAuth2 client (for Google login)
- `league/oauth2-google` 4.1.0 — Google OAuth2 provider
- NOT installed: `league/oauth2-server-bundle` — needed for being an OAuth2 server

## Architectural Decision: Built-in OAuth2 Server

Chosen approach: Build OAuth2 authorization server into Symfony using `league/oauth2-server-bundle`.

**Why not Keycloak**: Would add external dependency, user sync complexity, and operational overhead. The app already manages its own users.

**What league/oauth2-server-bundle provides**:
- Authorization code grant with PKCE
- Refresh token grant
- Client credentials grant
- Token revocation
- Configurable token lifetimes
- Doctrine entities for clients, access tokens, refresh tokens, auth codes

**What we need to add on top**:
- Protected Resource Metadata endpoint (RFC 9728) — JSON response at well-known URI
- Authorization Server Metadata endpoint (RFC 8414) — JSON response at well-known URI
- Client ID Metadata Document support (SHOULD) — validate URL-formatted client_ids
- Dynamic Client Registration endpoint (RFC 7591) — MAY, optional fallback
- Login/consent Twig template
- Proper `WWW-Authenticate` header on 401 responses (with `resource_metadata` and `scope`)
- Symfony firewall configuration for OAuth2 token validation on MCP/GraphQL routes
- Token audience validation (RFC 8707)

## Key Standards

- **OAuth 2.1** (draft-ietf-oauth-v2-1-13): Core authorization framework
- **RFC 9728**: Protected Resource Metadata (MUST)
- **RFC 8414**: Authorization Server Metadata discovery (MUST)
- **RFC 8707**: Resource Indicators — audience binding (MUST)
- **RFC 7636**: PKCE — Proof Key for Code Exchange (MUST, S256)
- **draft-ietf-oauth-client-id-metadata-document-00**: Client ID Metadata Documents (SHOULD)
- **RFC 7591**: Dynamic Client Registration (MAY)
- **RFC 7662**: Token Introspection (optional — can validate JWT locally instead)
