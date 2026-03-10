# Configuration

Hugin stores its configuration at `~/.config/hugin/config.toml`. Generate a default config with:

```bash
hugin init
```

View the current configuration:

```bash
hugin config show
```

## Proxy

```toml
[proxy]
listen_addr = "127.0.0.1:8080"     # Proxy listen address
api_addr = "127.0.0.1:8081"        # API / MCP server address
upstream_proxy = "socks5://127.0.0.1:9050"  # Route traffic through Tor, Burp, etc.
invisible_proxy = false             # Transparent proxy mode (no browser config needed)
additional_ports = [8443]           # Extra ports to listen on
```

## CA Certificate

```toml
[ca]
cert_path = "~/.config/hugin/Hugin-Proxy-CA.pem"
key_path = "~/.config/hugin/Hugin-Proxy-CA-key.pem"
cache_size = 1000                   # Per-host certificate cache size
```

## Storage

```toml
[storage]
db_path = "~/.config/hugin/hugin.db"
max_body_size = 10485760            # Max response body stored per flow (10 MB)
```

## Scope

```toml
[scope]
include_hosts = ["*.example.com"]   # Only capture matching hosts (empty = all)
exclude_hosts = ["*.analytics.com"] # Never capture matching hosts
```

Scope can also be configured in the Scopes view within the desktop UI.

## HTTP/2

```toml
[http2]
enabled = true
alpn_protocols = ["h2", "http/1.1"]
enable_server_push = true
max_concurrent_streams = 100
```

Force HTTP/1.1 for specific hosts:

```toml
[[http2.host_overrides]]
pattern = "legacy.example.com"
http_version = "ForceHttp11"
```

## Oastify (Out-of-Band Detection)

```toml
[oastify]
enabled = false
base_url = "https://oastify.eu"
domain = "oastify.eu"
api_token = "your-token"
poll_interval_ms = 5000
session_ttl_hours = 24
```

## API Authentication

```toml
[api]
auth_enabled = false
auth_username = "admin"
auth_password = "changeme"
auth_token = "bearer-token-here"
```

Enable authentication when exposing the API beyond localhost (e.g., with `hugin serve`).

## Telemetry

```toml
[telemetry]
enabled = false                     # Opt-in anonymous usage telemetry
```

Toggle from the CLI:

```bash
hugin config telemetry on
hugin config telemetry off
```

## External Tools

```toml
[tools]
xmass_api = "http://127.0.0.1:8080"
vectorsploit_hub = "http://127.0.0.1:50051"
subflow_path = "/usr/local/bin/subflow"
```
