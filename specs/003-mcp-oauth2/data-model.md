# Data Model: MCP OAuth2 User Authentication

## Entities

### Bundle-Managed Entities (league/oauth2-server-bundle)

These entities are provided by the bundle and stored in auto-generated tables. No custom code needed — the bundle manages their lifecycle.

#### OAuth2 Client (`oauth2_client`)

Represents an AI agent application (Claude Desktop, ChatGPT, VS Code, etc.).

| Field | Type | Description |
|-------|------|-------------|
| identifier | string(32) | Client ID (UUID or URL for CIMD clients) |
| name | string(128) | Human-readable name |
| secret | string(128), nullable | Client secret (null for public clients) |
| redirect_uris | text[] | Allowed redirect URIs |
| grants | text[] | Allowed grant types (authorization_code, refresh_token) |
| scopes | text[] | Allowed scopes (mcp:tools) |
| active | boolean | Whether client is active |
| allow_plain_text_pkce | boolean | Always false (S256 required) |

#### OAuth2 Access Token (`oauth2_access_token`)

| Field | Type | Description |
|-------|------|-------------|
| identifier | string(80) | Token identifier (JTI claim) |
| client | FK → oauth2_client | Issuing client |
| user_identifier | string, nullable | User email |
| scopes | text[] | Granted scopes |
| revoked | boolean | Whether token is revoked |
| expiry | datetime | Token expiration |

#### OAuth2 Refresh Token (`oauth2_refresh_token`)

| Field | Type | Description |
|-------|------|-------------|
| identifier | string(80) | Refresh token identifier |
| access_token | FK → oauth2_access_token | Associated access token |
| revoked | boolean | Whether token is revoked |
| expiry | datetime | Token expiration |

#### OAuth2 Authorization Code (`oauth2_authorization_code`)

| Field | Type | Description |
|-------|------|-------------|
| identifier | string(80) | Auth code identifier |
| client | FK → oauth2_client | Requesting client |
| user_identifier | string, nullable | User email |
| scopes | text[] | Requested scopes |
| revoked | boolean | Whether code is revoked |
| expiry | datetime | Code expiration (10 min) |

### App-Managed Entity

#### OAuth2AuthorizationContext (`oauth2_authorization_context`)

Stores the workspace/country/clinic context selected by the user during the OAuth consent flow. Linked to the authorization code identifier so context can be carried through to the access token.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | integer | PK, auto-generated (SEQUENCE) | Primary key |
| auth_code_identifier | string(80) | NOT NULL, indexed | Links to the bundle's auth code |
| user | FK → user | NOT NULL | The authenticated user |
| workspace | string(20), nullable | One of: project, exploitation, administration | Selected workspace |
| country_code | string(5), nullable | e.g., MA, SA | Selected country |
| clinic_id | integer, nullable | References clinic table | Selected clinic tenant |
| created_at | datetime | NOT NULL, default CURRENT_TIMESTAMP | When consent was granted |

**Relationships**:
- `user` → ManyToOne to existing `User` entity
- `clinic_id` → Soft reference to `Clinic` entity (no FK constraint needed — optional field)

**Lifecycle**:
1. Created when user clicks "Authorize" on the consent page (ApproveController)
2. Read by CustomAccessTokenRepository when generating access token JWT claims
3. Can be cleaned up periodically (contexts older than 30 days can be deleted)

## Entity Diagram

```
┌─────────────────────┐     ┌──────────────────────────┐
│   oauth2_client     │     │ oauth2_authorization_code │
│                     │◄────│                          │
│ identifier (PK)     │     │ identifier (PK)          │
│ name                │     │ client_id (FK)           │
│ secret              │     │ user_identifier          │
│ redirect_uris       │     │ scopes                   │
│ grants              │     │ expiry                   │
│ scopes              │     └──────────┬───────────────┘
│ active              │                │ auth_code_identifier
└─────────┬───────────┘                │
          │                 ┌──────────▼───────────────┐
          │                 │ oauth2_authorization_    │
          │                 │ context (APP-MANAGED)    │
          │                 │                          │
          │                 │ id (PK)                  │
          │                 │ auth_code_identifier     │
          │                 │ user_id (FK → user)      │
          │                 │ workspace                │
          │                 │ country_code             │
          │                 │ clinic_id                │
          │                 │ created_at               │
          │                 └──────────────────────────┘
          │
┌─────────▼───────────┐     ┌──────────────────────────┐
│ oauth2_access_token │────►│ oauth2_refresh_token     │
│                     │     │                          │
│ identifier (PK)     │     │ identifier (PK)          │
│ client_id (FK)      │     │ access_token_id (FK)     │
│ user_identifier     │     │ revoked                  │
│ scopes              │     │ expiry                   │
│ revoked             │     └──────────────────────────┘
│ expiry              │
└─────────────────────┘
```

## Access Token JWT Claims

The access token is a signed JWT (using the existing RSA keypair) with these claims:

| Claim | Type | Description |
|-------|------|-------------|
| aud | string | Resource server URI (e.g., `https://api.example.com`) — RFC 8707 |
| sub | string | User email |
| iat | integer | Issued at timestamp |
| exp | integer | Expiration timestamp (1 hour) |
| nbf | integer | Not before timestamp |
| jti | string | Token identifier |
| scopes | string[] | Granted scopes (`["mcp:tools"]`) |
| workspace | string, nullable | Default workspace from consent |
| country_code | string, nullable | Default country from consent |
| clinic_id | integer, nullable | Default clinic from consent |

The `workspace`, `country_code`, and `clinic_id` are custom private claims added by the `CustomAccessTokenRepository` decorator.

## Existing Entities (No Changes)

- **User**: Already has clinics, sectionCategories, environments (workspaces), clinicRestrictionEnabled, sectionRestrictionEnabled. No modifications needed.
- **ApiKey**: Kept for backward compatibility on `/custom/invoices/liasse` endpoint. No changes.
- **Clinic**: Referenced by clinic_id in authorization context. No changes.
