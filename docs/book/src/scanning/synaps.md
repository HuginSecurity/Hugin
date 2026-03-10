# Synaps Scanner

Synaps is Hugin's WASM-based vulnerability scanner. Modules are compiled to WebAssembly and executed in a sandboxed runtime (Wasmtime), providing safe, portable, and extensible scanning. Each module runs with a 1 billion instruction fuel limit and a 16 MB memory cap. A misbehaving or malicious module cannot hang the scanner or consume unbounded resources.

## How It Works

1. Modules are written in Rust and compiled to `wasm32-unknown-unknown`.
2. At scan time, Hugin loads modules and feeds them target information.
3. Each module decides whether to check a target (`should_check`) and performs its analysis (`check`).
4. Modules can make HTTP requests, DNS queries, raw TCP connections, WebSocket connections, and browser automation calls through the host runtime.
5. Results (findings) are collected and stored in the database.

## Module Lifecycle

```text
get_info()      -> Returns module metadata (name, severity, tags, CVE, CWE, CVSS)
should_check()  -> Returns true/false based on target characteristics
check()         -> Performs the actual vulnerability check
```

## Host Capabilities

Modules interact with the outside world exclusively through the `Context` trait. The host brokers all calls. Available capabilities:

- **HTTP** -- GET, POST, and custom requests with headers and bodies
- **DNS** -- A/AAAA/TXT queries
- **TLS** -- Certificate inspection
- **Raw TCP** -- Arbitrary TCP connections with timeout control
- **WebSocket** -- Connect, send, receive, close
- **OOB (Oastify)** -- Generate DNS/HTTP/SMTP/LDAP/FTP/SMB callback payloads and check for triggers
- **Headless browser** -- Navigate, inspect DOM, fill forms, click, screenshot
- **Inter-module data** -- Share data between producer and consumer modules
- **Logging** -- Debug, info, warn messages

## Module Workflows (Producer/Consumer)

Modules can form pipelines. A producer module sets `is_producer: true` in its metadata and stores data for downstream modules via `set_shared_data()`. Consumer modules declare dependencies and retrieve that data via `get_shared_data()` and `get_module_result()`. The runtime respects execution order: producers run first.

## Managing Modules

### CLI

```bash
# Download/update community modules
hugin scanner update

# List installed modules
hugin scanner list

# Install a specific module
hugin scanner install ai-gateway-detect

# Remove a module
hugin scanner remove example-module
```

### MCP Tool: `synaps`

The `synaps` MCP tool provides full module management and scanning. Requires a Pro license.

**Core actions:**

- `list` -- Show installed modules. Filter by `tags`, `severity`, `cve`.
- `info` -- Get detailed metadata for a specific module.
- `scan` -- Run one or more modules against a target.
- `validate` -- Check that a `.wasm` binary has valid exports.
- `stats` -- Module database statistics.
- `tags` -- List all tags across modules.
- `search` -- Find modules by keyword.

**Module-specific scans** (run a single specialized check):

- `scan_ai_gateway` -- AI gateway fingerprinting
- `scan_ai_ssrf` -- AI agent SSRF detection
- `scan_bare_lf` -- Bare LF HTTP request smuggling
- `scan_cache_poison` -- Cache poisoning
- `scan_charset_rce` -- Charset-based RCE
- `scan_fluentbit` -- Fluent Bit CVE detection
- `scan_graphql_intro` -- GraphQL introspection
- `scan_graphql_sub` -- GraphQL subscription endpoint checks
- `scan_grpc_web` -- gRPC-Web endpoint detection
- `scan_jwt_confusion` -- JWT algorithm confusion
- `scan_mass_assign` -- Mass assignment
- `scan_mqtt` -- MQTT protocol analysis
- `scan_nextjs_csrf` -- Next.js CSRF token bypass
- `scan_oidc` -- OIDC logout endpoint detection
- `scan_quic` -- QUIC protocol fingerprinting
- `scan_rust_http` -- HTTP differential response analysis
- `scan_rust_panic` -- Rust panic endpoint detection
- `scan_ssrf` -- Server-side request forgery
- `scan_vectordb` -- Vector database endpoint detection
- `scan_wcd` -- Web cache deception
- `scan_webtransport` -- WebTransport endpoint detection

## Community Modules

The [synaps-community](https://github.com/HuginSecurity/synaps-community) repository contains community-contributed modules organized by category:

- **web** -- Web framework vulnerabilities (Next.js, GraphQL, HTTP smuggling)
- **cloud** -- Cloud service misconfigurations (AI gateways, FluentBit CVEs)
- **api** -- API security issues (OAuth, OIDC)
- **injection** -- Injection vulnerabilities (charset RCE)
- **cve** -- Known CVE detection
- **tech** -- Technology fingerprinting (AI agents, vector DBs, MQTT, QUIC)

See [Community Modules](./community-modules.md) for details on installing and updating community modules, and [Module Development](./module-development.md) for the complete guide to writing your own.
