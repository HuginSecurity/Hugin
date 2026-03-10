# Plugin SDK

The Hugin Plugin SDK (`hugin-plugin-sdk`) is a Rust crate for building native plugins that integrate directly with the proxy engine. Unlike Lua extensions (which are scripts loaded at runtime), SDK plugins are compiled Rust code registered with Hugin's engine at startup.

The SDK is intentionally thin: it contains only traits and data types, no runtime machinery. Implementation details stay inside Hugin. This makes the SDK stable across minor versions.

## Getting Started

Add the SDK to your plugin's `Cargo.toml`:

```toml
[dependencies]
hugin-plugin-sdk = "0.1"
async-trait = "0.1"
```

## Hook Traits

The SDK provides four hook traits. Implement the ones relevant to your plugin.

### RequestHook

Intercept and optionally modify HTTP requests before they are forwarded to the target.

```rust
use hugin_plugin_sdk::{RequestHook, PluginContext, HttpRequestData, HookDecision};
use async_trait::async_trait;

pub struct MyRequestPlugin;

#[async_trait]
impl RequestHook for MyRequestPlugin {
    fn name(&self) -> &'static str { "my-request-plugin" }

    // Optional: set priority (lower = earlier). Default: 100
    fn priority(&self) -> u8 { 50 }

    async fn on_request(
        &self,
        ctx: &PluginContext,
        request: &mut HttpRequestData,
    ) -> HookDecision {
        // Modify the request in place
        request.headers.push(("X-Plugin".to_string(), "active".to_string()));

        // Return decision:
        HookDecision::Allow  // Forward (with modifications)
        // HookDecision::Drop   // Drop the request
        // HookDecision::Respond(response)  // Synthetic response
    }
}
```

### ResponseHook

Intercept and optionally modify HTTP responses before they reach the client.

```rust
use hugin_plugin_sdk::{ResponseHook, PluginContext, HttpRequestData, HttpResponseData, HookDecision};
use async_trait::async_trait;

pub struct MyResponsePlugin;

#[async_trait]
impl ResponseHook for MyResponsePlugin {
    fn name(&self) -> &'static str { "my-response-plugin" }

    async fn on_response(
        &self,
        ctx: &PluginContext,
        request: &HttpRequestData,
        response: &mut HttpResponseData,
    ) -> HookDecision {
        // Inspect or modify the response
        HookDecision::Allow
    }
}
```

### PassiveHook

Run passive analysis on stored flows. No outbound requests. Returns findings.

```rust
use hugin_plugin_sdk::{PassiveHook, PluginContext, PluginFinding, StoredFlow, FindingSeverity};
use async_trait::async_trait;

pub struct MyPassivePlugin;

#[async_trait]
impl PassiveHook for MyPassivePlugin {
    fn name(&self) -> &'static str { "my-passive-plugin" }

    async fn analyze(&self, ctx: &PluginContext, flow: &StoredFlow) -> Vec<PluginFinding> {
        let mut findings = vec![];

        if flow.request.url.contains("admin") {
            findings.push(
                PluginFinding::new(
                    "my-passive-plugin",
                    FindingSeverity::Low,
                    "Admin endpoint detected",
                )
                .with_description("Request to admin endpoint found in captured traffic")
                .with_flow(flow.id),
            );
        }

        findings
    }
}
```

### ActiveHook

Run active checks that can make outbound requests via `ctx.replay()`. Returns findings.

```rust
use hugin_plugin_sdk::{ActiveHook, PluginContext, PluginFinding, StoredFlow, FindingSeverity, HttpRequestData};
use async_trait::async_trait;

pub struct MyActivePlugin;

#[async_trait]
impl ActiveHook for MyActivePlugin {
    fn name(&self) -> &'static str { "my-active-plugin" }

    async fn scan(&self, ctx: &PluginContext, flow: &StoredFlow) -> Vec<PluginFinding> {
        // Replay the original request with a modification
        let mut modified = flow.request.clone();
        modified.url = format!("{}?test=1", modified.url);

        match ctx.replay(modified).await {
            Ok(response) if response.status == 500 => {
                vec![PluginFinding::new(
                    "my-active-plugin",
                    FindingSeverity::Medium,
                    "Server error on parameter injection",
                )]
            }
            _ => vec![],
        }
    }
}
```

## Data Types

### HttpRequestData

```rust
pub struct HttpRequestData {
    pub method: String,
    pub url: String,
    pub headers: Vec<(String, String)>,
    pub body: Option<Vec<u8>>,
}
```

### HttpResponseData

```rust
pub struct HttpResponseData {
    pub status: u16,
    pub headers: Vec<(String, String)>,
    pub body: Option<Vec<u8>>,
}
```

### StoredFlow

A complete request-response pair from the proxy history:

```rust
pub struct StoredFlow {
    pub id: uuid::Uuid,
    pub request: HttpRequestData,
    pub response: Option<HttpResponseData>,
    pub host: String,
    pub port: u16,
    pub is_tls: bool,
    pub timestamp: chrono::DateTime<chrono::Utc>,
}
```

### HookDecision

Returned by request/response hooks:

```rust
pub enum HookDecision {
    Allow,                        // Forward (with modifications applied)
    Drop,                         // Drop entirely
    Respond(HttpResponseData),    // Synthetic response (request hooks only)
}
```

### PluginFinding

Represents a security finding:

```rust
let finding = PluginFinding::new("plugin-id", FindingSeverity::High, "Title")
    .with_description("Detailed description")
    .with_flow(flow_id)  // Link to the flow that triggered it
    ;
```

Severity levels: `Critical`, `High`, `Medium`, `Low`, `Info`.

Confidence levels: `Certain`, `Firm`, `Tentative`.

### PluginContext

Provided to every hook at runtime. Gives access to:

- `ctx.replay(request)` -- Send an HTTP request and get the response (for active checks)
- `ctx.target_host()` -- Get the current scan target hostname
- `ctx.config()` -- Access per-plugin configuration (key-value pairs editable in the UI)

### PluginConfig

Per-plugin configuration accessible via `ctx.config()`:

```rust
let config = ctx.config();
let timeout = config.get_u64("timeout").unwrap_or(30);
let verbose = config.get_bool("verbose").unwrap_or(false);
let api_key = config.get_str("api_key").unwrap_or("");
```

## Plugin vs Lua Extension

Choose the right extension system for your use case:

**Use Lua extensions when:**
- You need to modify traffic in-flight (OnRequest/OnResponse with modifications)
- Quick prototyping without compilation
- Simple passive checks
- You want hot-reload without restarting Hugin

**Use the Plugin SDK when:**
- Performance-critical analysis (compiled Rust, no Lua overhead)
- Complex logic that benefits from Rust's type system and libraries
- Active scanning that requires concurrent HTTP requests
- You want to distribute a binary plugin

## Comparison

- **Lua extensions:** Interpreted, sandboxed (30s timeout, 64MB memory), hot-reloadable, 6 hook types, permission-gated API
- **Plugin SDK:** Compiled Rust, no sandbox overhead, registered at startup, 4 hook traits, full Rust ecosystem access
- Both can produce findings that appear in Hugin's scanner results
- Both are gated behind the Pro license
