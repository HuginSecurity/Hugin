# Architecture

Hugin is organized as a Rust workspace with 22 crates. Each crate has a focused responsibility. The entire system compiles to a single binary.

## Single Binary, Three Modes

The `hugin` binary (built from the `hugin-ui` crate) operates in three modes:

- **No arguments:** Launches the Dioxus desktop GUI
- **Subcommand:** Runs the corresponding CLI handler (e.g., `hugin start`, `hugin mcp`)
- **`hugin mcp`:** Starts the MCP server on stdio for AI assistant integration

## Crate Map

```
hugin-shared          Shared types: HttpFlow, HttpRequest, ScopeConfig, transforms, diff
hugin-core            MITM proxy engine (hudsucker/hyper), interceptor chain, WebSocket
hugin-store           SQLite persistence (sqlx): flows, findings, projects, scope, rules
hugin-engine          Active scanner (23 checks), passive scanner, plugin bus
hugin-service         Business logic: repeater, intruder, decoder, session, sequencer,
                      intelligence, updater, telemetry
hugin-api             Axum REST (~200 endpoints) + async-graphql API server
hugin-ui              Unified binary: Dioxus desktop GUI + CLI (31 views, 16+ subcommands)
hugin-mcp             MCP server (129 tool modules: 73 base + 56 vurl)
hugin-crawler         Web crawler (static + headless Chromium), JS analysis, form submission
hugin-oastify         OOB detection: DNS/HTTP/SMTP/LDAP/FTP/SMB listeners, payload generator
hugin-oastify-server  Standalone OOB callback server binary
hugin-extensions      Lua 5.4 plugin system (mlua), sandbox, permission model
hugin-ratrace         Race condition testing: barrier, last-byte, single-packet, 60+ modules
hugin-nerve           Passive response analysis: parameter discovery, pairs, parsers
hugin-scanner         WASM module system: feed compiler, DB loader, pattern matching
hugin-scanner-wasm    Wasmtime runtime for WASM scan modules
hugin-scanner-guest   Guest-side SDK for WASM module authors
hugin-scanner-cli     CLI for compiling/running/testing scan modules
hugin-browser         Browser automation: Chrome (CDP), Mullvad Browser (Marionette)
hugin-license         Ed25519 token verification, trial management, device fingerprinting
hugin-mobile          Android/iOS static analysis + Frida dynamic (library)
hugin-plugin-sdk      Plugin development SDK (traits + types, no runtime coupling)
```

## Dependency Graph

```
hugin-ui (single binary)
├── hugin-api (REST + GraphQL server)
│   ├── hugin-service (business logic)
│   │   ├── hugin-store (SQLite persistence)
│   │   │   └── hugin-shared (types)
│   │   ├── hugin-core (MITM proxy engine)
│   │   │   └── hugin-shared
│   │   └── hugin-shared
│   ├── hugin-engine (scanner)
│   │   ├── hugin-service
│   │   └── hugin-shared
│   ├── hugin-extensions (Lua plugins)
│   ├── hugin-ratrace (race testing)
│   ├── hugin-nerve (passive analysis)
│   ├── hugin-oastify (OOB detection)
│   └── hugin-browser (Chrome/Mullvad)
├── hugin-mcp (MCP server)
│   ├── hugin-api (shares AppState)
│   └── vurl (offensive modules)
└── hugin-scanner (WASM modules)
    └── hugin-scanner-wasm (Wasmtime runtime)
```

## Three Code Execution Systems

Hugin has three distinct code execution systems. Each serves a different purpose.

### 1. Native ActiveCheck (hugin-engine)

The built-in scanner. 41 active vulnerability checks compiled into the binary. Users do not write these.

- **Trait:** `ActiveCheck` in `hugin-engine/src/scanner/active.rs`
- **Registration:** `ScanExecutor::register_default_checks()` in `executor.rs`
- **Capabilities:** Payloads + analysis logic. The `ScanExecutor` handles HTTP sending, insertion point extraction, transform chain re-encoding, differential/blind detection, OOB callbacks, concurrency, rate limiting.
- Not sandboxed (native Rust code). Not extensible by users at runtime.

### 2. WASM Modules -- Synaps (hugin-scanner)

Community scanner modules. Third-party Rust code compiled to WASM, sandboxed via Wasmtime.

- **Guest SDK:** `hugin-scanner-guest/src/imports.rs` defines the `Context` trait with all host capabilities
- **Host runtime:** `hugin-scanner-wasm/src/host.rs` implements host functions
- **Capabilities:** HTTP, raw TCP, DNS, TLS inspection, WebSocket, browser automation, Oastify OOB, inter-module pipelines, SBB pattern matching, file read
- **Sandbox:** WASM isolation, fuel-based execution limits (1 billion instructions), 16 MB memory cap
- **Loading:** SBB database via `ModuleLoader` in `hugin-scanner/src/db/loader.rs`
- The WASM compilation friction is intentional -- it provides a strong sandbox guarantee for untrusted community code

Users install modules via `hugin scanner install <name>`.

### 3. Lua Extensions (hugin-extensions)

User customization scripts. No compilation required.

- **API:** `hugin-extensions/src/lua_api.rs`
- **Hooks:** `OnRequest`, `OnResponse`, `OnScanResult`, `OnFlowCapture`, `PassiveCheck`, `ActiveCheck`
- **Unique capability:** Only path that can modify traffic in-flight (change URL, headers, body, status, or drop)
- **Permissions:** `ReadFlows`, `ModifyFlows`, `NetworkAccess`, `FileSystem`, `SystemCommands`
- **Sandbox:** Lua debug hooks (10K instruction interval), 64 MB memory, 30s timeout, permission gating
- **Discovery:** Auto-scans extensions directory for `extension.json` manifests

## Key Data Types

- **`HttpFlow`** -- A request-response pair with metadata (timing, flags, annotations, project). Defined in `hugin-shared`.
- **`HuginStore`** -- SQLite persistence layer. All flows, findings, scope, rules, projects. In-memory mode for tests.
- **`AppState`** -- Axum shared state. Contains the store, proxy handle, broadcast channels, all service instances. Shared across API, MCP, and UI via `Arc`.
- **`ScanExecutor`** -- Scanner engine. Concurrent check execution with semaphore, rate limiting, pause/resume.
- **`IntruderAttack`** -- Fuzzer run. Payload generators, processors, HTTP send, grep match/extract.
- **`Project`** -- Workspace isolation. Flows, scope, findings, rules, repeater data. Exportable as `.huginproject`.

## Desktop UI Architecture

The desktop app uses Dioxus and renders natively. Zero TCP -- all API calls go through `tower::oneshot()` on the in-process Axum Router, avoiding JSON serialization overhead for local operations.

`BrowserMap = Arc<RwLock<HashMap<u16, BrowserSlot>>>` -- shared across API, MCP, and UI. `BrowserSlot` holds either a Chrome manager (CDP) or a Mullvad session (Marionette).

## MCP Architecture

The MCP server (`hugin-mcp`) uses `Arc<AppState>` directly -- zero HTTP, zero client abstraction. It shares the same service layer, store, and browser sessions as the GUI and REST API.

Communication is over stdio using the rmcp library. Responses are capped at 12,000 characters to avoid filling up the LLM context window.

## Data Flow

```
Browser/Client
    │
    ▼
┌──────────────────┐
│   hugin-core     │  MITM Proxy Engine
│  (interceptor    │  ─── intercept queue (if intercept enabled)
│   chain)         │  ─── Lua extension hooks (OnRequest/OnResponse)
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│   hugin-store    │  SQLite: persist flow
│                  │  ─── Lua extension hooks (OnFlowCapture)
└──────┬───────────┘
       │
       ▼
┌──────────────────┐
│  hugin-engine    │  Passive scanner (background)
│  hugin-nerve     │  Parameter analysis, reflection detection
│  hugin-extensions│  Lua PassiveCheck hooks
└──────────────────┘
       │
       ▼
┌──────────────────┐
│  hugin-api       │  REST + GraphQL (external consumers)
│  hugin-mcp       │  MCP tools (AI assistants)
│  hugin-ui        │  Desktop GUI (Dioxus)
└──────────────────┘
```

## Build

```bash
# Full workspace
cargo build --release

# Single binary (GUI + CLI)
cargo build --release --bin hugin

# All tests
cargo test --workspace

# API integration tests
cargo test -p hugin-api --test integration_tests
```

Linux build requires: `libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, `libayatana-appindicator3-dev`.

WASM modules build separately:

```bash
cd synaps-modules/example
cargo build --target wasm32-unknown-unknown --release
```

## External Dependencies

- **vurl** -- Private repo at `../vurl`, referenced as `vurl = { path = "../vurl" }`. CI needs `VURL_PAT` secret.
- **SQLite** -- Embedded via sqlx. No external database server needed.
- **Chromium** -- Optional, for headless crawling and browser automation. Feature-gated.
