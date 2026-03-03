# API Contracts: OAuth2 Endpoints

## Discovery Endpoints (Public, No Auth)

### GET `/.well-known/oauth-protected-resource`

**Purpose**: Protected Resource Metadata (RFC 9728). MCP clients fetch this to discover the authorization server.

**Response** `200 OK`:
```json
{
  "resource": "https://api.example.com",
  "authorization_servers": ["https://api.example.com"],
  "scopes_supported": ["mcp:tools"]
}
```

### GET `/.well-known/oauth-authorization-server`

**Purpose**: Authorization Server Metadata (RFC 8414). MCP clients fetch this to discover endpoints and capabilities.

**Response** `200 OK`:
```json
{
  "issuer": "https://api.example.com",
  "authorization_endpoint": "https://api.example.com/oauth/authorize",
  "token_endpoint": "https://api.example.com/oauth/token",
  "scopes_supported": ["mcp:tools"],
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "token_endpoint_auth_methods_supported": ["none", "client_secret_post"],
  "code_challenge_methods_supported": ["S256"],
  "client_id_metadata_document_supported": true
}
```

---

## Authorization Flow Endpoints

### GET `/oauth/authorize`

**Purpose**: Authorization endpoint. Validates OAuth params and redirects the browser to the React frontend consent page.

**Query Parameters**:
| Parameter | Required | Description |
|-----------|----------|-------------|
| client_id | Yes | Client identifier (string or URL for CIMD) |
| redirect_uri | Yes | Where to redirect after authorization |
| response_type | Yes | Must be `code` |
| code_challenge | Yes | PKCE code challenge (S256) |
| code_challenge_method | Yes | Must be `S256` |
| state | Yes | CSRF protection state value |
| scope | No | Defaults to `mcp:tools` |
| resource | No | RFC 8707 resource indicator (MCP server URI) |

**Response** `302 Found`:
- **Success**: Redirects to `{FRONTEND_URL}/oauth/consent?{all_params}`
- **Error** (invalid redirect_uri): `400 Bad Request` JSON error
- **Error** (other): Redirects to `{redirect_uri}?error={code}&error_description={msg}&state={state}`

### POST `/oauth/approve`

**Purpose**: Called by React frontend after user consents. Issues an authorization code.
**Auth**: Lexik JWT (user's existing frontend session token)

**Request Body** `application/json`:
```json
{
  "client_id": "claude-desktop",
  "redirect_uri": "http://localhost:3000/oauth/callback",
  "response_type": "code",
  "code_challenge": "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM",
  "code_challenge_method": "S256",
  "state": "xyz123",
  "scope": "mcp:tools",
  "resource": "https://api.example.com",
  "workspace": "exploitation",
  "country_code": "MA",
  "clinic_id": 42
}
```

**Response** `200 OK`:
```json
{
  "redirect_url": "http://localhost:3000/oauth/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=xyz123"
}
```

**Error Responses**:
- `401 Unauthorized`: JWT invalid or missing
- `400 Bad Request`: Invalid OAuth parameters
- `403 Forbidden`: User not authorized for requested scope

### POST `/oauth/deny`

**Purpose**: Called by React frontend when user denies authorization.
**Auth**: Lexik JWT

**Request Body** `application/json`:
```json
{
  "client_id": "claude-desktop",
  "redirect_uri": "http://localhost:3000/oauth/callback",
  "state": "xyz123"
}
```

**Response** `200 OK`:
```json
{
  "redirect_url": "http://localhost:3000/oauth/callback?error=access_denied&error_description=The+resource+owner+denied+the+request&state=xyz123"
}
```

### POST `/oauth/token`

**Purpose**: Token endpoint (provided by league/oauth2-server-bundle). Exchanges auth code for tokens or refreshes tokens.

**Authorization Code Exchange**:
```
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=http://localhost:3000/oauth/callback
&client_id=claude-desktop
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
&resource=https://api.example.com
```

**Response** `200 OK`:
```json
{
  "token_type": "Bearer",
  "expires_in": 3600,
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "def50200b3c7b8..."
}
```

**Refresh Token Exchange**:
```
POST /oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=def50200b3c7b8...
&client_id=claude-desktop
&resource=https://api.example.com
```

**Response** `200 OK`:
```json
{
  "token_type": "Bearer",
  "expires_in": 3600,
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refresh_token": "def50200a1d2e3..."
}
```

**Error Responses**:
- `400 Bad Request`: Invalid grant, invalid code_verifier, invalid refresh_token
- `401 Unauthorized`: Invalid client credentials

---

## MCP/GraphQL Error Responses

### 401 Unauthorized (No Token or Invalid Token)

When an MCP or GraphQL request lacks a valid token:

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://api.example.com/.well-known/oauth-protected-resource", scope="mcp:tools"
Content-Type: application/json

{
  "status": "error",
  "message": "Authentication required"
}
```

### 403 Forbidden (Insufficient Scope)

```
HTTP/1.1 403 Forbidden
WWW-Authenticate: Bearer error="insufficient_scope", scope="mcp:tools"
Content-Type: application/json

{
  "status": "error",
  "message": "Insufficient scope"
}
```

---

## React Frontend Endpoints

### GET `/oauth/consent` (React route, not API)

**Purpose**: React consent page. Displays client name, requested scopes, and workspace/country/clinic selectors.

**URL Parameters** (forwarded from Symfony `/oauth/authorize`):
- All OAuth parameters from the authorize request
- Parsed by the React component from `useSearchParams()`

**Behavior**:
- If user not logged in → redirects to `/auth/login?returnUrl=/oauth/consent?{params}`
- If logged in → shows consent form with Authorize/Deny buttons
- Authorize → `POST /oauth/approve` with JWT + context → `window.location.href` to returned redirect_url
- Deny → `POST /oauth/deny` → `window.location.href` to returned redirect_url
