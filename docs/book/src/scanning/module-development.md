# Writing Synaps WASM Modules

Synaps modules are Rust crates that compile to `wasm32-unknown-unknown` and run inside Hugin's sandboxed WASM runtime. Each module is an isolated, independently deployable vulnerability check. The sandbox enforces a 1 billion instruction fuel limit and a 16 MB memory cap.

## Project Setup

```bash
cargo new --lib my-check
cd my-check
```

Set the crate type in `Cargo.toml`:

```toml
[lib]
crate-type = ["cdylib"]

[dependencies]
hugin-synaps-guest = { path = "../../hugin-scanner-guest" }
serde_json = "1"
```

Add a `.cargo/config.toml` to set the default target:

```toml
[build]
target = "wasm32-unknown-unknown"
```

## Module Structure

Every module needs three functions wired together with the `synaps_module!` macro: `get_info`, `check`, and optionally `should_check`.

```rust,ignore
use hugin_synaps_guest::prelude::*;

fn get_info() -> ModuleInfo {
    ModuleInfo {
        id: "my-vuln-check".into(),
        name: "My Vulnerability Check".into(),
        author: "yourname".into(),
        description: "Detects XYZ misconfiguration".into(),
        severity: Severity::High,
        tags: vec!["web".into(), "misconfig".into()],
        cve: Some("CVE-2024-1234".into()),
        cwe: Some(vec!["CWE-16".into()]),
        cvss: Some(7.5),
        references: vec!["https://example.com/advisory".into()],
        dependencies: vec![],
        is_producer: false,
    }
}

fn should_check(target: &TargetInfo) -> bool {
    target.scheme == "https"
}

fn check<C: Context>(ctx: &C, target: &TargetInfo) -> Result<CheckResult, CheckError> {
    let resp = ctx.http_get("/api/status")?;

    if resp.status == 200 && resp.contains("debug_mode: true") {
        return Ok(CheckResult::vulnerable(Confidence::High)
            .with_evidence(Evidence::http_response(resp.body_str().unwrap_or("")))
            .with_message("Debug mode enabled on production endpoint"));
    }

    Ok(CheckResult::not_vulnerable())
}

synaps_module!(
    info: get_info,
    check: check,
    should_check: should_check
);
```

## The Context Trait

The `Context` trait is the module's interface to the host. It provides every capability the module needs without direct network or filesystem access -- the runtime brokers all calls.

### HTTP

```rust,ignore
// Simple GET
let resp = ctx.http_get("/path")?;

// Custom request with headers
let req = HttpRequest::get("/api/data")
    .with_header("X-Custom", "value")
    .with_header("Accept", "application/json");
let resp = ctx.http_request(&req)?;

// POST with body
let resp = ctx.http_post("/submit", b"{\"key\":\"value\"}".to_vec())?;

// Inspect response
println!("Status: {}", resp.status);
println!("Body: {}", resp.body_str().unwrap_or(""));
println!("Header: {}", resp.header("content-type").unwrap_or(""));
```

### DNS and TLS

```rust,ignore
let dns = ctx.dns_query(&DnsQuery::a("target.example.com"))?;
let first_ip = dns.first_value();

let tls = ctx.tls_info("target.example.com", 443)?;
let cert = tls.certificate.as_ref();
```

### Raw TCP

```rust,ignore
let req = TcpRequest::new("target.example.com", 8080, b"PING\r\n".to_vec())
    .with_timeout(3000);
let resp = ctx.tcp_request(&req)?;
let data = resp.data_str().unwrap_or("");
```

### WebSocket

```rust,ignore
let conn = ctx.ws_connect("wss://target.example.com/ws")?;
ctx.ws_send_text(conn, r#"{"action":"ping"}"#)?;
let msg = ctx.ws_recv(conn, 5000)?;
ctx.ws_close(conn)?;
```

### OOB (Oastify) Payloads

```rust,ignore
let payload = ctx.oastify_dns(Some("my-check"))?;
// Inject payload.payload into the target
// ...
// Wait and check for callback
if ctx.oastify_was_triggered(&payload.correlation_id) {
    return Ok(CheckResult::vulnerable(Confidence::Confirmed)
        .with_message("OOB DNS interaction received"));
}
```

### Headless Browser

```rust,ignore
ctx.browser_navigate("https://target.example.com/login", 2000)?;
let dom = ctx.browser_get_dom()?;
if dom.has_form_action("/submit") {
    ctx.browser_input("#username", "test")?;
    ctx.browser_click("#submit")?;
}
let screenshot = ctx.browser_screenshot()?;
```

### Logging

```rust,ignore
ctx.debug("Checking endpoint /api");
ctx.info("Found suspicious header");
ctx.warn("Unexpected response status");
```

## Severity and Confidence

`Severity` maps to CVSS scores when `from_cvss()` is used: Critical (9.0+), High (7.0+), Medium (4.0+), Low (0.1+). Set it in `get_info`.

`Confidence` on `CheckResult` has four levels:

- `Confirmed` -- OOB callback proof or definitive evidence. Use only when you have Oastify or equivalent confirmation.
- `High` -- Direct response evidence (error messages, math evaluation, file contents).
- `Medium` -- Differential signals (response differs from baseline).
- `Low` -- Heuristic or indirect signals.

## Module Workflows (Producer/Consumer)

Modules can form pipelines. A producer module sets `is_producer: true` and stores data for downstream modules:

```rust,ignore
// Producer module
ctx.set_shared_data("discovered_api_key", &api_key)?;
```

Consumer modules declare dependencies and retrieve that data:

```rust,ignore
// In get_info
dependencies: vec!["my-producer-module".into()],

// In check
let api_key = ctx.get_shared_data("discovered_api_key")?;
let result = ctx.get_module_result("my-producer-module")?;
if let Some(r) = result {
    if r.is_vulnerable() {
        // Build on the producer's finding
    }
}
```

## Building

```bash
cargo build --target wasm32-unknown-unknown --release
```

The output is at `target/wasm32-unknown-unknown/release/my_check.wasm`.

## Testing Without WASM

The guest SDK ships a `MockContext` for native unit tests. The macros compile correctly for native targets and use `MockContext` automatically:

```rust,ignore
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_module_info() {
        let info = get_info();
        assert_eq!(info.severity, Severity::High);
        assert!(!info.id.is_empty());
    }

    #[test]
    fn test_should_check_https_only() {
        let target = TargetInfo {
            scheme: "https".into(),
            host: "example.com".into(),
            ..Default::default()
        };
        assert!(should_check(&target));

        let http_target = TargetInfo { scheme: "http".into(), ..target };
        assert!(!should_check(&http_target));
    }
}
```

Run native tests with:

```bash
cargo test
```

## Installing a Built Module

Copy the `.wasm` file into the Hugin modules directory or use the `synaps` MCP tool with the `scan` action pointing to a local path. The runtime validates the WASM magic bytes and exports before loading.
