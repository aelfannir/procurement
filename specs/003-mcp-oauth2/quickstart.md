# Quickstart: MCP OAuth2 User Authentication

## Prerequisites

- Docker Compose environment running (`docker compose up -d`)
- Existing RSA keypair at `config/jwt/private.pem` and `config/jwt/public.pem`
- React frontend app (`procurement-app`) running at `http://localhost:3000`

## Setup Steps

### 1. Install Dependencies

```bash
docker compose exec php composer require league/oauth2-server-bundle nyholm/psr7 symfony/psr-http-message-bridge
```

### 2. Generate OAuth2 Encryption Key

```bash
docker compose exec php vendor/bin/generate-defuse-key
```

Add the output to `.env`:
```
OAUTH2_ENCRYPTION_KEY=def000...
```

### 3. Configure Environment Variables

Add to `.env`:
```
OAUTH2_ENCRYPTION_KEY=<generated-key>
FRONTEND_URL=http://localhost:3000
OAUTH2_ISSUER=http://localhost:84
```

### 4. Run Database Migration

```bash
docker compose exec php bin/console doctrine:migrations:diff
docker compose exec php bin/console doctrine:migrations:migrate
```

### 5. Create a Test OAuth2 Client

```bash
docker compose exec php bin/console league:oauth2-server:create-client \
    "mcp-test-client" \
    --grant-type=authorization_code \
    --grant-type=refresh_token \
    --scope=mcp:tools \
    --redirect-uri=http://127.0.0.1/callback
```

### 6. Verify Metadata Endpoints

```bash
# Protected Resource Metadata (RFC 9728)
curl http://localhost:84/.well-known/oauth-protected-resource

# Authorization Server Metadata (RFC 8414)
curl http://localhost:84/.well-known/oauth-authorization-server
```

Both should return JSON with the correct endpoint URLs.

### 7. Test with Claude Desktop

Add MCP server configuration to Claude Desktop:
```json
{
  "mcpServers": {
    "procurement": {
      "url": "http://localhost:84/mcp",
      "transport": "streamable-http"
    }
  }
}
```

Claude Desktop will:
1. Send request to `/mcp` → get `401` with `WWW-Authenticate` header
2. Fetch `/.well-known/oauth-protected-resource` → discover auth server
3. Fetch `/.well-known/oauth-authorization-server` → discover endpoints
4. Open browser to `/oauth/authorize` → redirect to React consent page
5. After user authorizes → exchange code for token at `/oauth/token`
6. Use token for MCP requests

## Key Configuration Files

| File | Purpose |
|------|---------|
| `config/packages/league_oauth2_server.yaml` | OAuth2 bundle configuration |
| `config/packages/security.yaml` | Firewall configuration (OAuth2 authenticator) |
| `config/routes/oauth.yaml` | OAuth endpoint routes (no /custom prefix) |
| `.env` | OAUTH2_ENCRYPTION_KEY, FRONTEND_URL, OAUTH2_ISSUER |

## Troubleshooting

**"Invalid token" on MCP requests**: Verify the RSA public key matches between Lexik JWT and the OAuth2 bundle. Both should use `config/jwt/public.pem`.

**401 without WWW-Authenticate header**: Check that `OAuth2ChallengeListener` is registered and the `oauth2Issuer` parameter is correctly wired.

**Consent page redirects to login infinitely**: Ensure the `/oauth/consent` route is registered as a top-level route in `AppRoutes.tsx` (not inside PrivateRoutes/PublicRoutes).

**"Unsupported grant type"**: Verify that `enable_auth_code_grant: true` and `enable_refresh_token_grant: true` are set in `league_oauth2_server.yaml`.
