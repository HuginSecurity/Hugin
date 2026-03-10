# Configuration Reference

Hugin is configured via a TOML file at `~/.config/hugin/config.toml`. Generate a default config with `hugin init`. If the file does not exist, Hugin uses built-in defaults.

## File Locations

- **Config file:** `~/.config/hugin/config.toml`
- **CA certificate:** `~/.config/hugin/Hugin-Proxy-CA.pem`
- **CA private key:** `~/.config/hugin/Hugin-Proxy-CA-key.pem`
- **SQLite database:** `~/.config/hugin/hugin.db`
- **Extensions directory:** `~/.config/hugin/extensions/`

## Configuration Sections

### `[proxy]`

Controls the MITM proxy engine.

```toml
[proxy]
# Proxy listen address and port
listen_addr = "127.0.0.1:8080"

# API server address and port (optional, defaults to 127.0.0.1:8081)
api_addr = "127.0.0.1:8081"

# Upstream proxy URL for chaining (e.g., Tor, Burp, corporate proxy)
# upstream_proxy = "socks5://127.0.0.1:9050"

# Enable transparent (invisible) proxy mode — no browser config required
invisible_proxy = false

# Additional ports to bind the proxy on (e.g., for HTTPS interception)
additional_ports = []

# Per-host custom TLS certificate overrides
# [[proxy.per_host_certs]]
# host = "api.example.com"
# cert_path = "/path/to/cert.pem"
```

### `[ca]`

Controls the Certificate Authority used for TLS interception.

```toml
[ca]
# Path to the CA certificate (PEM format)
cert_path = "~/.config/hugin/Hugin-Proxy-CA.pem"

# Path to the CA private key (PEM format)
key_path = "~/.config/hugin/Hugin-Proxy-CA-key.pem"

# Number of generated certificates to cache in memory
cache_size = 1000
```

### `[storage]`

Controls data persistence.

```toml
[storage]
# SQLite database path
db_path = "~/.config/hugin/hugin.db"

# Maximum HTTP body size to store (bytes). Bodies exceeding this are truncated.
max_body_size = 10485760  # 10 MB
```

### `[scope]`

Initial scope configuration. Scope is usually managed at runtime via the UI or MCP tools.

```toml
[scope]
# Hosts to include in scope (supports glob patterns)
include_hosts = ["*.example.com", "api.target.com"]

# Hosts to exclude from scope
exclude_hosts = ["analytics.example.com"]
```

### `[api]`

API server authentication. See [Authentication](../api/authentication.md) for details.

```toml
[api]
# Enable authentication (required when binding to non-loopback addresses)
auth_enabled = false

# Basic Auth credentials
auth_username = "admin"
auth_password = "your-password"

# Static Bearer token (alternative to Basic Auth)
auth_token = "your-api-token"
```

### `[http2]`

HTTP/2 protocol settings for the proxy.

```toml
[http2]
# Enable HTTP/2 support
enabled = true

# ALPN protocols to advertise during TLS handshake
alpn_protocols = ["h2", "http/1.1"]

# Enable HTTP/2 server push handling
enable_server_push = true

# Maximum concurrent streams per connection
max_concurrent_streams = 100

# Initial window size for flow control (bytes)
initial_window_size = 65535

# Maximum frame size (bytes, must be 16384-16777215)
max_frame_size = 16384

# Enable HPACK header compression
enable_hpack = true

# Maximum header list size (bytes)
max_header_list_size = 16384
```

Per-host HTTP version overrides:

```toml
[[http2.host_overrides]]
pattern = "legacy.example.com"
http_version = "ForceHttp11"

[[http2.host_overrides]]
pattern = "*.modern.com"
http_version = "ForceHttp2"
```

HTTP version override values:

- `PreferHttp2` -- Prefer HTTP/2, fallback to HTTP/1.1
- `ForceHttp2` -- Force HTTP/2 only (fail if not supported)
- `ForceHttp11` -- Force HTTP/1.1 only
- `Auto` -- Automatic negotiation (default)

### `[oastify]`

Out-of-band interaction detection configuration. Connects to a remote Oastify server for DNS/HTTP callback tracking.

```toml
[oastify]
# Enable Oastify integration
enabled = false

# Base URL for Oastify API
base_url = "https://oastify.eu"

# DNS callback domain (if different from base_url host)
# domain = "oastify.eu"

# API token for authentication (optional)
# api_token = "your-token"

# Default session name for payload tracking
session = "default"

# Poll interval in milliseconds (minimum: 100)
poll_interval_ms = 5000

# Session time-to-live in hours (1-168)
session_ttl_hours = 24
```

### `[tools]`

Paths and endpoints for external tool integrations.

```toml
[tools]
# Path to nerve binary (auto-detected from PATH if not set)
# nerve_path = "/usr/local/bin/nerve"

# Path to ghostcheck binary
# ghostcheck_path = "/usr/local/bin/ghostcheck"

# Path to rattrace binary
# rattrace_path = "/usr/local/bin/rattrace"

# XMass API endpoint
xmass_api = "http://127.0.0.1:8080"

# VectorSploit Hub gRPC endpoint
vectorsploit_hub = "http://127.0.0.1:50051"

# Path to subflow binary (auto-detected from PATH if not set)
# subflow_path = "/usr/local/bin/subflow"
```

### `[telemetry]`

Anonymous telemetry configuration. See [Telemetry](telemetry.md) for details.

```toml
[telemetry]
# Master switch (off by default)
enabled = false

# Telemetry backend endpoint
endpoint = "https://telemetry.hugin.nu/v1/events"

# Flush interval in seconds (minimum: 10)
flush_interval_secs = 300

# Maximum events buffered before eager flush (1-10000)
max_batch_size = 100

# Extra tags added to every event batch
# [telemetry.tags]
# team = "infra"
```

## Full Default Config

Generate a complete default config:

```bash
hugin init
```

This writes `~/.config/hugin/config.toml` with all default values and inline comments.

## Runtime Configuration

Many settings can be changed at runtime without restarting Hugin:

- **Scope:** Via MCP (`scope` tool), REST API (`/api/scope`), or the UI
- **Upstream proxy:** Via MCP (`settings` tool), REST API (`/api/settings/upstream-proxy`), or the UI
- **HTTP/2 settings:** Via MCP (`settings` tool) or REST API (`/api/settings/http2`)
- **Intercept rules:** Via MCP (`rules` tool) or REST API (`/api/rules`)
- **Extensions:** Via MCP (`extensions` tool), CLI (`hugin plugin`), or REST API (`/api/extensions`)
- **Telemetry:** Via CLI (`hugin config telemetry on/off`)

Changes to the proxy listen address, CA certificate paths, and storage paths require a restart.

## Environment Variables

- `HUGIN_CONFIG` -- Override the config file path
- `HUGIN_LOG` -- Set log level (e.g., `debug`, `info`, `warn`, `error`)
- `RUST_LOG` -- Fine-grained log filtering (e.g., `hugin_core=debug,hugin_mcp=info`)
