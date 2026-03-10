# Community Modules

Community modules are Synaps WASM checks distributed through the Hugin module registry. They are sandboxed by the Wasmtime runtime -- each module runs with a 1 billion instruction fuel limit and 16 MB memory cap. A misbehaving or malicious module cannot affect the host process.

## How Modules Are Stored

Installed modules live on disk in the Hugin modules directory. Each module is represented by two files:

```text
{modules_dir}/
  {module-id}.wasm    -- compiled WASM binary
  {module-id}.json    -- metadata (version, checksum, install timestamp)
```

The metadata file records the module ID, name, version, category, severity, SHA256 checksum, and installation time. If the `.json` file is present but the `.wasm` is missing, the entry is skipped at load time.

## The Catalog Format

The registry catalog is a JSON document served from a remote URL (typically a GitHub raw URL). Each entry includes all fields needed for display, filtering, and downloading. The `wasm_filename` is used to construct the download URL: `{base_url}/modules/{category}/{wasm_filename}`.

## Installing Modules

Use the `synaps` MCP tool or the Hugin UI scanner panel. The install flow:

1. Fetch the remote catalog from the configured registry URL.
2. Compare against locally installed versions.
3. Download the WASM binary for each new or updated module.
4. Compute the SHA256 of the downloaded bytes and verify it against the expected checksum from the catalog (when a checksum is provided).
5. Write the `.wasm` and `.json` files to the modules directory.

If a module with the same ID already exists, the install overwrites both files.

### CLI

```bash
hugin scanner update          # Fetch catalog and install new/updated modules
hugin scanner list            # List installed modules
hugin scanner install <name>  # Install a specific module
hugin scanner remove <name>   # Remove a module
```

## Updating Modules

The diff logic compares catalog entries against locally installed modules by ID and version string. A module is marked for update when the catalog version differs from the installed version. New modules (IDs not present locally) are also included in the update list and tagged `[NEW]`. Updated modules show their previous version alongside the new version as `[UPD]`.

## Filtering at Load Time

When the scanner loads modules for a run, it filters by:

- Module ID (explicit allowlist via `enabled_checks` in scan config -- empty means all).
- Whether the module's declared `applicable_locations` intersect with the target's insertion points.
- The `should_check` export -- the module itself can reject a target based on scheme, technologies, or metadata before any network request is made.

## Module Dependencies

Some modules declare dependencies on other modules via the `dependencies` array in their `ModuleInfo`. The runtime respects execution order: producer modules run first, then consumers can read their extracted data via `get_module_result()` and `get_shared_data()`. A module marked `is_producer: true` is expected to write shared data for downstream modules.

## Available Community Modules

The current synaps-modules directory includes the following modules:

- `example` -- Directory listing detection (reference implementation)
- `nextjs-csrf` -- Next.js CSRF token bypass detection
- `oidc-logout-detect` -- OIDC logout endpoint detection
- `quic-fingerprint` -- QUIC protocol fingerprinting
- `webtransport-detect` -- WebTransport endpoint detection
- `bare-lf-smuggling` -- Bare LF HTTP request smuggling
- `rust-panic` -- Rust panic endpoint detection
- `rust-http-diff` -- HTTP differential response analysis
- `graphql-subscription` -- GraphQL subscription endpoint checks
- `charset-rce` -- Charset-based RCE detection
- `mqtt-analyze` -- MQTT protocol analysis
- `grpc-web-detect` -- gRPC-Web endpoint detection
- `ai-gateway-detect` -- AI gateway fingerprinting
- `fluentbit-cve` -- Fluent Bit CVE detection
- `ai-agent-ssrf` -- AI agent SSRF detection
- `oauth-misconfig` -- OAuth misconfiguration checks
- `vector-db-detect` -- Vector database endpoint detection

## Integrity Verification

Every module download computes a SHA256 hash of the raw WASM bytes. When the recommended download path is used, the computed hash must match the expected checksum from the catalog exactly. A mismatch returns a checksum error and the module is not installed.

## Removing a Module

Removing a module deletes both its `.wasm` binary and `.json` metadata file. If only one file exists, it is still removed. Attempting to remove a module that does not exist returns a not-found error.
