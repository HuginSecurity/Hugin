# Authentication

Hugin has two distinct authentication systems:

1. **API authentication** -- protects the HTTP API from unauthorized access when exposed on a network interface
2. **License authentication** -- determines which features are available based on your account tier

## API Authentication

### When Is It Needed?

When running locally (bound to `127.0.0.1`), authentication is disabled by default. Anyone with local access can use the API.

When bound to a network interface (`0.0.0.0` or a specific IP), authentication should be enabled. Hugin logs a warning if it detects a non-loopback bind address without authentication enabled.

### Authentication Methods

The API supports three authentication methods, checked in order:

**1. Bearer Token (config-based)**

A static token defined in the config file. Best for scripts and automation.

```bash
curl http://host:8081/api/flows \
  -H "Authorization: Bearer your-static-token"
```

Config:

```toml
[api]
auth_enabled = true
auth_token = "your-static-token"
```

**2. Database Access Tokens (`hgn_*`)**

Dynamic tokens stored in the database. Created and managed via CLI or API. Best for team collaboration where each member gets their own token.

```bash
# Create a token
hugin token create --label "alice"
# Output: hgn_a1b2c3d4e5f6...

# Use it
curl http://host:8081/api/flows \
  -H "Authorization: Bearer hgn_a1b2c3d4e5f6..."

# List tokens
hugin token list

# Revoke
hugin token revoke hgn_a1b2c3d4e5f6...
```

Token management API:

```
GET    /api/tokens          List all tokens
POST   /api/tokens/create   Create token (optional: label)
DELETE /api/tokens/{token}  Revoke token
```

**3. HTTP Basic Auth**

Username/password authentication. Useful when integrating with tools that only support Basic Auth.

```bash
curl http://host:8081/api/flows \
  -u "admin:secretpassword"
```

Config:

```toml
[api]
auth_enabled = true
auth_username = "admin"
auth_password = "secretpassword"
```

### Unauthenticated Endpoints

The health check endpoint is always accessible without authentication, allowing monitoring tools and load balancers to verify the service is running:

```
GET /api/health    Always accessible (no auth)
```

### Configuration

Full API authentication config in `~/.config/hugin/config.toml`:

```toml
[api]
# Master switch for authentication
auth_enabled = false

# Basic Auth credentials
auth_username = "admin"
auth_password = "your-password"

# Static Bearer token
auth_token = "your-api-token"
```

### Headless Server Mode

When running `hugin serve` for team collaboration, authentication is enabled by default (since it binds to `0.0.0.0`). The `--no-auth` flag disables it but should only be used on trusted networks:

```bash
# Default: auth enabled, bind to 0.0.0.0
hugin serve --port 8080 --api-port 8081

# DANGEROUS: disable auth on trusted network only
hugin serve --no-auth
```

## License Authentication

License authentication controls feature access, not API access. It is separate from API auth.

### Tiers

- **Community (free):** Proxy, scanner, intruder, sequencer, repeater, decoder, basic MCP tools
- **Pro (paid):** Vurl MCP tools (49 offensive modules), Synaps WASM modules, RatRace, Lua extensions, multi-project, collaboration

### Account Setup

```bash
# Set your account ID (format: HGN-XXXXXXXX-XXXXXXXX-XXXXXXXX)
hugin account set HGN-XXXXXXXX-XXXXXXXX-XXXXXXXX

# Check status
hugin account show

# Remove
hugin account clear
```

API:

```
GET  /api/license/status    Current license status and tier
POST /api/license/account   Set account ID
```

### License Verification

License tokens are Ed25519 signed and verified locally. The license state refreshes periodically in the background. No phone-home for every API call -- verification is done at startup and on refresh intervals.

## Security Recommendations

- Always enable API auth when binding to non-loopback addresses
- Use `hgn_*` database tokens for team members (revocable, auditable)
- Keep the static Bearer token for automated systems where token rotation is impractical
- When running `hugin serve`, ensure the API port is not publicly exposed without auth
- Use TLS termination (e.g., Caddy or nginx reverse proxy) if exposing Hugin over the network
