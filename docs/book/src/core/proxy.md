# Proxy

Hugin's proxy is a full MITM (man-in-the-middle) HTTP and WebSocket proxy. It sits between your browser and the target application, decrypting TLS traffic, capturing every request and response, and feeding them to all other tools in real time.

## Getting Started

1. Start Hugin. The proxy listens on `127.0.0.1:8080` by default.
2. Install the CA certificate (see below).
3. Configure your browser to use `127.0.0.1:8080` as its HTTP/HTTPS proxy.
4. Browse the target application. All traffic appears in the History view.

## How It Works

For plain HTTP, the proxy captures requests and responses directly.

For HTTPS, the flow is:

1. Your browser sends a CONNECT tunnel request for the target host.
2. Hugin completes the TLS handshake with the browser using a dynamically generated certificate for that host.
3. Hugin establishes a separate TLS connection to the upstream server.
4. Every request and response is captured as an HTTP flow and stored in the database.
5. The flow is broadcast to the UI, scanner, and any loaded plugins immediately.

## HTTP/2 Support

HTTP/2 is enabled by default. The proxy negotiates the protocol via ALPN during the upstream TLS handshake. When the server speaks HTTP/2, Hugin uses a persistent connection pool keyed by host. Pseudo-headers (`:method`, `:path`, `:authority`, `:scheme`) are captured alongside regular headers so HTTP/2 flows are fully inspectable.

HTTP/2 can be disabled in configuration if a target misbehaves with it.

## CA Certificate

On first launch, Hugin generates an RSA CA key pair and writes it to disk. You install the CA certificate once so your browser trusts the proxy's per-host certificates.

**Installation:**

- **macOS** -- double-click the certificate, add to Keychain, mark as "Always Trust"
- **Firefox** -- Settings > Privacy & Security > Certificates > View Certificates > Import
- **Chrome/Chromium** -- follows the OS trust store on macOS and Linux

The CA cert PEM is served at `http://127.0.0.1:8081/api/ca.pem` for easy download. You can also export it via the CLI:

```
hugin ca --print
hugin ca --output ~/ca.crt
```

## Scope Filtering

Scope controls what gets captured and stored. Four modes are available:

- **CaptureAll** (default) -- every flow is stored regardless of host
- **InScopeOnly** -- only flows matching at least one include pattern are stored
- **OutOfScopeOnly** -- flows matching exclude patterns are dropped before storage
- **CaptureAllTagOOS** -- all flows are stored, but out-of-scope flows are tagged

Scope patterns can match on host, URL prefix, or regular expression. Scope is updated at runtime without restarting the proxy; changes take effect on the next request.

Out-of-scope traffic is always forwarded transparently -- the browser still gets its responses, they just are not stored.

## Upstream Proxy Chaining

Hugin can route traffic through an upstream proxy. Supported types:

- HTTP and HTTPS proxies
- SOCKS4 and SOCKS5 proxies
- Per-host routing rules with wildcard patterns and priority ordering
- Basic authentication for proxies that require credentials
- A built-in `tor` preset routing through `socks5://127.0.0.1:9050`

Per-host rules are evaluated in priority order (higher numbers first). If no rule matches, Hugin falls back to the global proxy or connects directly.

## WebSocket Detection

When the proxy sees an `Upgrade: websocket` request, it hands the connection to the WebSocket manager. The HTTP 101 handshake is recorded as a normal flow, and all subsequent frames are captured separately. See [WebSocket](./websocket.md) for details.

## Plugin and Extension Hooks

Every proxied flow passes through the plugin bus before being stored. Registered plugins can modify, flag, or drop requests and responses. Lua extensions can hook `OnRequest` and `OnResponse` to modify traffic in-flight.

## Flow Recording

Flow recording can be paused and resumed without stopping the proxy. When paused, the proxy still forwards traffic but does not write flows to the database. This is useful when you want to reduce noise while performing a specific test.

## Configuration

Proxy settings are stored in `~/.config/hugin/config.toml`:

```toml
[proxy]
listen_addr = "127.0.0.1:8080"

[ca]
cert_path = "~/.config/hugin/Hugin-Proxy-CA.pem"
key_path  = "~/.config/hugin/Hugin-Proxy-CA-key.pem"

[http2]
enabled        = true
alpn_protocols = ["h2", "http/1.1"]
```

## CLI

```
hugin start                        # Start proxy on default port
hugin start --port 9090            # Start on custom port
hugin start --bind 0.0.0.0         # Bind to all interfaces
hugin status                       # Show proxy status
hugin ca --print                   # Print CA certificate to stdout
```

## MCP Tool

**`proxy_status`** -- proxy infrastructure info.

Actions: `health`, `status`, `ca_cert`

**`scope`** -- manage proxy scope.

Actions: `get`, `set_mode`, `add_pattern`, `remove_pattern`, `update`, `export`, `import`, `save_preset`, `load_preset`, `list_presets`, `delete_preset`, `from_sitemap`
