# MCP Integration

Hugin ships a built-in [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) server that exposes 126 tools to AI assistants. This is the primary interface for using Hugin with Claude Code, Claude Desktop, and any other MCP-compatible client.

The MCP server runs in-process and shares state directly with the proxy engine via `Arc<AppState>` -- zero HTTP overhead, zero serialization boundaries. Every tool call goes through the same service layer the GUI and REST API use.

## Setup

### Claude Code

```bash
claude mcp add hugin -- hugin mcp
```

Or add to your `.mcp.json`:

```json
{
  "mcpServers": {
    "hugin": {
      "command": "hugin",
      "args": ["mcp"]
    }
  }
}
```

### Claude Desktop

Add to `claude_desktop_config.json` (macOS: `~/Library/Application Support/Claude/`, Linux: `~/.config/Claude/`):

```json
{
  "mcpServers": {
    "hugin": {
      "command": "hugin",
      "args": ["mcp"]
    }
  }
}
```

### Standalone Mode

The MCP server can also be started alongside the proxy:

```bash
hugin start --mcp
```

Or as a standalone process:

```bash
hugin mcp
```

In standalone mode, the MCP server starts its own proxy and API internally. No separate `hugin start` is needed.

## Architecture

The MCP server (`hugin-mcp` crate) uses the [rmcp](https://crates.io/crates/rmcp) library and communicates over stdio (stdin/stdout). It has direct access to:

- `HuginService` -- business logic layer (flows, scanner, intruder, etc.)
- `HuginStore` -- SQLite persistence
- `BrowserMap` -- CDP/Marionette browser automation
- `ScanExecutor` -- vulnerability scanner engine
- All vurl offensive modules (Pro license)

Responses are automatically capped at 12,000 characters to prevent context window overflow. Large bodies are truncated with size indicators by default (use `include_body=true` for full content).

Flow IDs accept both full UUIDs and short prefixes (minimum 4 characters), similar to git short hashes.

## License Tiers

- **Community (free):** 42 base tools covering proxy control, flow management, scanning, intruder, repeater, decoder, sequencer, and more.
- **Pro (paid):** 49 additional vurl offensive tools for parser differentials, WAF evasion, request smuggling, AI agent exploitation, cloud SSRF, and more. Plus 35 additional advanced tools (ratrace, synaps, intelligence, etc.).

## Tool Reference

All 126 MCP tools grouped by category. Each tool uses an `action` parameter to select the operation.

---

### Flow Management (4 tools)

**list_flows** -- List captured HTTP flows with optional filters for method, host, URL, status code, and flagged state.

Parameters: `method`, `host`, `url_contains`, `status_code`, `flagged`, `limit`, `offset`

**get_flow** -- Get full details of a captured HTTP flow including request, response, headers, body, and persisted intelligence (nerve findings, client-side findings, response intel). Bodies are truncated by default -- use `include_body=true` for full bodies. Accepts full UUIDs or short prefixes (>= 4 chars).

Parameters: `id`, `include_body`, `max_body_bytes`

**search_flows** -- Search/list captured flows. Provide query to filter by URL text, or omit to list all flows.

Parameters: `query`, `limit`

**annotate_flow** -- Annotate a flow: flag, unflag, highlight, comment, or delete.

Parameters: `id`, `action` (flag, unflag, highlight, comment, delete), `comment`, `color`

---

### Proxy Control (3 tools)

**intercept** -- MITM intercept control: intercept, inspect, modify, forward or drop HTTP requests and responses in real time. Also controls flow recording (history capture).

Actions: `status`, `toggle`, `recording`, `list`, `get`, `forward`, `drop`, `forward_all`, `drop_all`, `responses`, `response_get`, `response_forward`, `response_drop`, `responses_forward_all`, `responses_drop_all`

**scope** -- Proxy scope management: control which hosts/URLs are captured.

Actions: `get`, `set_mode` (CaptureAll/InScopeOnly/OutOfScopeOnly/CaptureAllTagOOS), `add_pattern`, `remove_pattern`, `update`, `export`, `import`, `save_preset`, `load_preset`, `list_presets`, `delete_preset`, `from_sitemap`

**proxy_status** -- Proxy infrastructure info: health check, statistics, CA certificate.

Actions: `health`, `status`, `ca_cert`

---

### Request Replay (2 tools)

**repeater** -- HTTP request repeater: replay requests with optional modifications. Set `browser_port` to route through a running browser's JS `fetch()` for real TLS fingerprint (JA3/JA4), bypassing WAF bot detection (Akamai BMP, DataDome, Cloudflare).

Actions: `send`, `history`, `queue`, `queue_status`, `raw_send`, `batch_raw_send`, `repeat`, `repeat_flow`, `batch_send`, `list_queue`, `cancel`, `clear_queue`, `proxy_status`, `compare`

**comparer** -- Response comparer: compare HTTP responses for blind vulnerability detection.

Actions: `compare`, `blind_detect`, `similarity`

---

### Vulnerability Scanning (9 tools)

**scanner** -- Active vulnerability scanner: scan flows for security issues (SQLi, XSS, SSRF, etc.).

Actions: `start` (provide flow_ids), `status`, `findings`, `checks`, `cancel`, `pause`, `resume`, `clear`, `list_scans`, `get_scan`, `delete_scan`, `get_finding`, `delete_finding`, `update_finding`, `audit_items`, `create_finding`, `set_finding_status`, `add_finding_flow`, `remove_finding_flow`, `list_finding_flows`, `add_finding_tag`, `remove_finding_tag`, `list_finding_tags`

**scanner_audit_items** -- Get scanner audit items: group vulnerability findings by check ID for a specific scan.

**authz** -- Authorization matrix scanner (Autorize-like): replay captured flows with different auth contexts (admin, user, no-auth), diff responses, flag access control issues.

Actions: `scan`, `findings`, `export`

**idor** -- IDOR scanner: extract parameterized IDs from captured flows, swap them with attacker's auth, compare responses.

Actions: `scan`, `extract`, `findings`

**sqli** -- SQL injection scanner: test every parameter from captured flows for SQLi using error-based, time-based blind, and boolean-based blind techniques.

Actions: `scan`, `test_param`, `payloads`

**xss** -- Reflected XSS scanner: inject probes into every parameter from captured flows, detect reflection without encoding, try context-appropriate payloads.

Actions: `scan`, `test_param`, `payloads`

**pathtraversal** -- Path traversal fuzzer: fuzz file-related parameters with 8 encoding variants (plain, URL, double-URL, UTF-8 overlong, null byte, backslash, semicolon, mixed).

Actions: `scan`, `test_param`, `payloads`

**pipeline** -- Security pipeline orchestrator: run all scanners in sequence (flow_analysis, authz, idor, sqli, pathtraversal, xss). Aggregate findings, generate reports.

Actions: `run`, `findings`, `report`

**synaps** -- Synaps WASM vulnerability scanner: run security checks using hot-swappable WASM modules.

Core actions: `list`, `info`, `scan`, `validate`, `stats`, `tags`, `search`. Module-specific scans: `scan_ai_gateway`, `scan_oidc`, `scan_vectordb`, `scan_graphql_sub`, `scan_quic`, `scan_grpc_web`, `scan_mqtt`, `scan_charset_rce`, `scan_bare_lf`, `scan_rust_panic`, `scan_fluentbit`, `scan_rust_http`, `scan_ai_ssrf`, `scan_nextjs_csrf`, `scan_webtransport`, `scan_ssrf`, `scan_wcd`, `scan_cache_poison`, `scan_graphql_intro`, `scan_jwt_confusion`, `scan_mass_assign`

---

### Fuzzing and Discovery (5 tools)

**intruder** -- Intruder attack automation: fuzz parameters, brute force, enumerate. Use `{}` markers for injection points.

Actions: `start`, `list`, `get`, `status`, `pause`, `resume`, `cancel`, `delete`, `results`, `grep_local`, `grep_data`, `processing`, `export`

**discover** -- Content discovery: brute-force directories, files, and backup patterns. Auto-calibrates wildcard detection. Set `detect_sinks=true` to flag DOM XSS sinks in response bodies.

Actions: `run`, `wildcard_check`

**ffuzzer** -- FUZZ keyword fuzzer: replaces FUZZ (and FUZ2-FUZ9 for multi-keyword) in URL/headers/body with wordlist entries. Two modes: `http` (raw TCP/TLS, fast) and `browser` (Mullvad Browser via Marionette, bypasses TLS fingerprinting).

**param_discover** -- Parameter discovery: fuzz parameter names to find hidden/undocumented params. Batch testing (10 params/request) then individual verification via response diffing.

Actions: `run`, `check`

**api_spec** -- API specification auto-discovery: probe known spec paths (OpenAPI/Swagger, GraphQL introspection, WSDL/SOAP, Docker registry) and parse into structured routes and parameters.

Actions: `discover`, `parse`

---

### Intelligence and Analysis (5 tools)

**intelligence** -- Cross-flow intelligence engine: aggregate analysis of captured HTTP traffic.

Params: `params`, `param_search`, `param_stats`, `param_endpoint`. Routes: `routes`, `route_detail`. Reflections: `reflections`, `reflection_candidates`, `reflection_stats`. Endpoints: `endpoints`, `endpoint_detail`, `endpoint_stats`. Security: `rollups`, `rollup_summary`, `rollup_host`. Nerve: `nerve_findings`, `nerve_stats`. Client-side: `client_side`, `client_side_stats`. Response: `response_intel`, `response_intel_stats`. Gold: `gold`. Backfill: `analyze`. Purge: `purge_intel`. Auth: `auth_diff`.

**paramhunter** -- Parameter signal analysis: maps HTTP parameter names/values to 21 vulnerability categories with 700+ regex signals and confidence levels.

Actions: `analyze`, `analyze_flows`, `categories`, `info`, `stats`

**flow_analysis** -- Flow security analyzer: scan captured proxy traffic for postMessage handlers and DOM sinks.

Actions: `postmessage`, `dom_sinks`, `all`

**fingerprint** -- Technology fingerprinting: passive detection of web technologies from HTTP headers and response content.

Actions: `categories`, `signatures`, `security_headers`, `recommend`, `analyze_headers`, `profile`

**scan_optimizer** -- AI-powered scan optimization: intelligent priority scoring and adaptive scanning.

Actions: `analyze`, `recommend_checks`, `profile`, `stats`, `learn`

---

### Race Condition Testing (1 tool)

**ratrace** -- Race condition testing engine: test for TOCTOU, double-spend, rate limit bypass vulnerabilities.

Core actions: `test`, `detect`, `quick`, `limit`, `batch`, `sessions`, `session`, `cancel`, `result`, `discover`, `param_hunt`, `endpoints`, `scan`, `scan_findings`, `ws`, `microservice`, `cache_race`, `state_fuzz`, `orchestrate`, `report`, `graphql_subscription`, `oidc_logout`. Engine config: `engine_config`, `engine_timing`, `engine_graphql`. Enhancements: `enhance_fuzz`, `enhance_lock`, `enhance_protocol`, `enhance_lag`. Detection: `detect_db`, `detect_node`, `detect_thresholds`. Advanced: `blind`, `multisession`, `patterns`, `report_custom`.

---

### Encoding and Decoding (2 tools)

**decoder** -- Decoder/encoder for security testing. 622 polyglot payloads across 66 contexts (XSS, SQLi, NoSQL, command injection, SSTI, XXE, SSRF, path traversal, and many more).

Actions: `encode`, `decode`, `chain`, `analyze`, `reencode`, `jwt_decode`, `jwt_forge`, `polyglot`, `operations`

**smart_decode** -- Smart decoder: auto-detect encoding chains (base64, URL, HTML entities, hex, JWT, unicode escapes, gzip, double-URL) and decode layer by layer.

Actions: `detect`, `auto_decode`, `detect_and_decode`, `encodings`

---

### Site Map and Crawling (3 tools)

**site_map** -- Site map: explore captured HTTP traffic by host and path.

Actions: `hosts`, `hosts_simple`, `host`, `paths`, `flows`, `search`, `stats`, `tree`, `export`, `path_detail`

**crawler** -- Web crawler: discover pages, forms, and URLs by crawling a target site.

Actions: `start`, `stop`, `pause`, `resume`, `status`, `urls`, `export`

**vurl_crawl** -- Standalone web crawler using Vurl HTTP engine. Works without Hugin proxy running. Supports multiple routing modes (direct, mullvad, hugin, custom proxy).

Actions: `modes`, `check_proxy`, `start`, `stop`, `pause`, `resume`, `status`, `urls`, `export`, `list`, `delete`

---

### Session and Authentication (4 tools)

**session** -- Session and authentication management: track tokens, create login macros, auto-refresh sessions.

Actions: `tokens`, `status`, `list_macros`, `create_macro`, `get_macro`, `delete_macro`, `execute_macro`, `refresh`

**hugin_cookie_jar** -- Cookie jar management: view, edit, delete, filter, import/export cookies from proxy sessions.

Actions: `list`, `get`, `set`, `delete`, `clear`, `domains`, `export`, `import`, `from_flows`, `expired`, `purge_expired`

**hugin_macros** -- Session macros: record, edit, and replay multi-step request sequences for session maintenance and authentication.

Macro actions: `list_macros`, `get_macro`, `create_macro`, `update_macro`, `delete_macro`, `run_macro`, `test_macro`. Rule actions: `list_rules`, `create_rule`, `update_rule`, `delete_rule`, `enable_rule`, `disable_rule`

**settings** -- Proxy and HTTP/2 settings: configure upstream proxy, per-host rules, presets, HTTP/2 options.

Actions: `get_proxy`, `set_proxy`, `clear_proxy`, `list_rules`, `add_rule`, `remove_rule`, `replace_rules`, `test_proxy`, `preset` (tor/burp/mullvad/disable), `get_http2`, `set_http2`

---

### Browser Automation (2 tools)

**browser** -- Browser automation: launch Chrome (CDP) or Mullvad Browser (Marionette) through Hugin proxy, navigate to URLs, capture all traffic, and run full recon pipeline.

Actions: `launch`, `browse`, `navigate`, `crawl`, `status`, `stop`, `exec_js`, `source`, `screenshot`, `new_tab`, `switch_tab`, `close_tab`, `list_tabs`

**screenshot** -- Screenshot capture for bug bounty PoC evidence. Uses the existing browser session from BrowserMap.

Actions: `capture`, `capture_flow`, `capture_ui`, `record_start`, `record_stop`, `list`

---

### OOB Detection (2 tools)

**oastify** -- Rust-native OOB server: local out-of-band interaction detection with DNS and HTTP listeners.

Actions: `start`, `stop`, `status`, `domain`, `stats`, `generate_payload`, `batch_payloads`, `all_payloads`, `list_payloads`, `get_payload`, `interactions`, `interaction`, `delete_interaction`, `acknowledge`, `acknowledge_bulk`

**vurl_oastify** -- Oastify OOB callback tracking: central hub for all tools (intruder, vsploit, scanner, manual). Connect to oastify server, register payloads with unique tracked markers, poll/sync for callbacks.

Actions: `connect`, `disconnect`, `status`, `generate`, `register_batch`, `payloads`, `interactions`, `sync`, `stats`, `commands`, `generate_local`

---

### Token Analysis (1 tool)

**sequencer** -- Token randomness sequencer: capture tokens from repeated requests and analyze their randomness quality (FIPS statistical tests).

Actions: `capture`, `tokens`, `stop`, `analyze`, `list`, `delete`, `status`, `compare`, `export`

---

### Project Management (2 tools)

**project** -- Project isolation: manage per-target project profiles with isolated scope, flows, and recon data.

Actions: `create`, `list`, `get`, `update`, `delete`, `activate`, `deactivate`, `archive`, `scope`, `stats`, `assign_flows`, `export`, `templates`, `create_from_template`, `fingerprint`, `import_scope`, `policy_get`, `policy_set`, `policy_search`. Legacy: `save`, `load`, `recent`

**hugin_environment** -- Manage named environments and variables: create environments, set/get/delete variables, activate.

---

### Reporting and Export (3 tools)

**reporting** -- Generate security reports in various formats.

Actions: `sarif`, `summary`, `html`, `markdown`, `csv`, `formats`, `executive_summary`, `templates`, `list_templates`, `save_template`, `delete_template`, `generate`, `issue_select`

**hugin_exports** -- Export captured flows and project data in JSON, CSV, or HAR format.

**output_store** -- Searchable output storage for large tool results. Stores outputs from intruder, repeater, scanner in SQLite with FTS5 full-text search.

Actions: `list`, `search`, `get`, `delete`, `delete_old`, `stats`

---

### Organization and Workflow (6 tools)

**dashboard** -- Dashboard overview: get event log, active tasks, and aggregated stats.

Actions: `events`, `tasks`, `stats`

**events** -- Event log management: view, filter, and manage system events.

Actions: `list`, `recent`, `stats`, `clear`, `delete_before`

**hugin_organizer** -- Organizer: save, categorize, annotate, and search interesting HTTP requests for triage and reporting.

Actions: `list`, `get`, `save`, `update`, `delete`, `search`, `categories`, `tags`, `export`, `bulk_tag`, `bulk_delete`

**hugin_collections** -- Manage collections: curated bundles of HTTP requests with annotations.

Actions: `list`, `get`, `create`, `update`, `delete`, `add_flow`, `add_raw`, `remove_item`, `reorder`, `annotate_item`, `export`, `import`, `duplicate`, `share`

**hugin_scheduler** -- Manage scheduled scan jobs: list, create, update, delete, trigger, and view run history.

**hugin_workflows** -- Manage event-driven workflows: create triggers and actions for automated flow processing.

---

### Rules and Filters (3 tools)

**rules** -- Manage intercept rules for request/response filtering and actions. Rules can match by host, path, method, headers, body content.

Actions: `list`, `get`, `create`, `update`, `delete`, `groups_list`, `groups_create`, `groups_delete`

**hugin_logger_filters** -- Logger filter management: save/load filter presets for HTTP history, manage capture filter rules for conditional logging.

Preset actions: `list_presets`, `get_preset`, `save_preset`, `delete_preset`, `apply_preset`. Capture filter actions: `list_capture_filters`, `create_capture_filter`, `update_capture_filter`, `delete_capture_filter`, `enable_capture_filter`, `disable_capture_filter`

**hugin_bambda** -- Bambda: Lua filter expressions for flow tables. Write inline Lua code to filter, search, and transform captured proxy traffic.

Actions: `filter`, `transform`, `test`, `presets`, `save_preset`, `delete_preset`

---

### DOM/Client-Side Security (4 tools)

**hugin_taint** -- Browser-based DOM XSS taint analysis: launch headless Chrome to trace data flow from user-controlled sources to dangerous sinks.

Actions: `scan`, `analyze_flow`, `sources`, `sinks`

**hugin_cors** -- CORS misconfiguration scanner: test `Access-Control-Allow-Origin` / `Access-Control-Allow-Credentials` behavior with 9 origin probes.

Actions: `scan`, `scan_flow`, `techniques`

**hugin_upload** -- File upload vulnerability scanner: test upload endpoints with 16 extension variants, 6 content-type mismatches, double extensions, null byte injection, and polyglot file generation.

Actions: `scan`, `techniques`, `generate_polyglot`

**vurl_postmessage** -- postMessage security scanner: static analysis of JS chunks for postMessage handlers, origin validation quality, and event.data flow into dangerous sinks.

Actions: `analyze`, `patterns`, `scan_url`

---

### JS Analysis (4 tools)

**vurl_js_sinks** -- JS chunk DOM sink finder: download JavaScript chunks from a page, scan for dangerous sinks (innerHTML, eval, document.write) and track controllable data sources.

Actions: `analyze`, `sinks`, `scan_url`

**vurl_js_endpoints** -- JS endpoint extractor: extract API routes, URLs, fetch/XHR calls, GraphQL operations, and WebSocket endpoints from JavaScript.

Actions: `analyze`, `scan_url`, `patterns`, `mine_params`

**vurl_endpointer** -- EndPointer: endpoint behavioral profiler. Probes discovered API endpoints to build behavioral profile cards across 20 tiers.

Actions: `probe`, `batch`, `methods`, `auth`, `profile`

**vurl_csp_nonce** -- CSP nonce leak detector: detect nonce values in CSP headers, meta tags, and script attributes. Test nonce reuse across multiple requests.

Actions: `detect`, `headers`, `reuse`

---

### Next.js / React (3 tools)

**vurl_hydration** -- Generate Next.js RSC hydration hijacking payloads. Exploits `__NEXT_DATA__`, RSC chunks, multipart prototype pollution.

Actions: `generate`, `categories`

**vurl_nextjs_rsc** -- Next.js RSC (React Server Components) analyzer: parse RSC flight data, detect component tree leaks, test nonce reuse, cache poisoning.

Actions: `analyze`, `nonce`, `cache_poison`, `reflection`

**vurl_nextjs_middleware** -- Next.js middleware bypass tester: test header-based, path normalization, locale prefix, and `_next/data` route bypasses.

Actions: `test`, `headers`, `paths`, `locale`

---

### HTTP Smuggling (2 tools)

**vurl_smuggle** -- HTTP request smuggling payloads and detection.

Actions: `payloads` (CL.TE/TE.CL), `te_variants`, `host`, `keep_alive`, `probe_0cl`, `probe_cl0`, `double_desync`, `early_gadgets`, `full_scan`, `header_casing`

**vurl_harvest** -- SmuggleHarvester: continuous request smuggling attack daemon. Rotates through confirmed techniques, captures victim data.

Actions: `start`, `stop`, `status`, `list`, `results`, `techniques`

---

### SSRF and Cloud (4 tools)

**vurl_cloud** -- Generate cloud metadata SSRF payloads for AWS, GCP, Azure, DigitalOcean, Alibaba, Oracle, Kubernetes, Docker. Includes IMDSv1/v2.

**vurl_ssrf_detect** -- SSRF detection engine: analyze responses for SSRF indicators and perform timing-based blind SSRF detection.

Actions: `analyze`, `indicators`, `timing_baseline`, `timing_analyze`, `timing_compare`, `is_blocked`

**vurl_k8s** -- Generate Kubernetes SSRF payloads (IngressNightmare, storage controller, admission webhooks).

Actions: `generate`, `types`

**vurl_redirect** -- Open redirect chain payloads for SSRF.

Actions: `generate`, `params`, `redirectors`

---

### WAF Evasion and Fingerprinting (6 tools)

**vurl_evade** -- Generate WAF evasion payloads using encoding tricks, case manipulation, null bytes, unicode normalization.

**vurl_waf_evasion** -- WAF evasion techniques: encoding mutations, protocol quirks, Unicode tricks, chunked abuse.

Actions: `evade`, `categories`

**vurl_mirage** -- Browser fingerprint spoofing and WAF evasion via Mirage module. Generate coherent browser profiles (TLS JA3/JA4, HTTP/2 AKAMAI fingerprints).

Actions: `bypasses`, `profiles`, `headers`, `oracle`, `techniques`

**vurl_sni** -- TLS SNI manipulation for WAF bypass. Exploits disconnect between TLS SNI (routing) and HTTP Host header (application).

Actions: `payloads`, `openssl`, `ncat`, `python`, `curl`, `raw`, `all`

**vurl_fingerprint** -- Parser fingerprinting via probe requests. Identifies target parser behavior.

Actions: `probes`, `all_probes`, `categories`, `signatures`

**vurl_charset** -- Generate charset-based RCE payloads using Best-Fit/Worst-Fit encoding attacks (Orange Tsai research).

Actions: `generate`, `command`, `path`, `curl`, `encodings`, `attack_types`

---

### URL Parsing and Mutation (3 tools)

**vurl_compare** -- Compare how different URL parsers interpret a URL. Detects parser differentials exploitable for SSRF bypass, open redirect, and path traversal.

**vurl_hunt** -- Hunt for parser differential vulnerabilities by generating URL mutations and comparing parser outputs.

**vurl_mutator** -- URL mutation engine for parser differential hunting. 15+ mutator strategies.

Actions: `mutate`, `chains`, `crlf`, `nfkc`, `strategies`

---

### HTTP Client (4 tools)

**vurl_http** -- Send HTTP request with full control over method, headers, body, timeouts, redirects, and proxy. Keep `max_body_size` under 15000 to avoid token overflow.

**vurl_http_raw** -- Send raw HTTP request string for smuggling attacks, malformed requests, or protocol-level testing.

**vurl_http_compare** -- Compare HTTP responses from two URLs to detect behavioral differences, timing variations, and content changes.

**vurl_chain** -- Chain HTTP requests with variable extraction. Each step can extract values from responses (JSON path or regex) and use them in subsequent requests via `{{variable}}` placeholders.

---

### Response Diffing (2 tools)

**vurl_diff** -- Response diffing engine for 0-day detection. Compares HTTP responses with 1-byte precision and 10ms timing resolution.

Actions: `compare`, `quick`, `batch`, `hash`, `timing`, `config`, `severity_score`

**vurl_diffing** -- Response diffing for 0-day detection: compare responses for byte-level and timing differences.

Actions: `compare`, `timing`, `config`

---

### Protocol-Level Attacks (6 tools)

**vurl_grpc** -- Generate gRPC gateway differential attack payloads. Exploits parsing differences between gRPC gateways and backends.

**vurl_grpc_matrix** -- Get gRPC gateway vs backend compatibility matrix showing known differential risks.

**vurl_h2** -- HTTP/2 single-packet race condition attacks. Uses frame multiplexing for microsecond-precision race conditions (James Kettle's research).

Actions: `payloads`, `curl`, `turbo_intruder`, `templates`, `frame_types`, `config`, `analyze`

**vurl_rust_http** -- Generate Rust HTTP parser differential payloads for request smuggling. Based on RUSTSEC-2020-0008, CVE-2021-32715, CVE-2025-32094.

Actions: `generate`, `templates`, `matrix`, `endpoints`, `cves`

**vurl_hopbyhop** -- Generate hop-by-hop header attack payloads for proxy bypass.

**vurl_quic** -- Generate HTTP/3 and QUIC attack payloads: 0-RTT replay, connection ID manipulation, QPACK table poisoning, Alt-Svc injection.

Actions: `generate`, `curl`, `detect`, `types`

---

### Authentication Bypass (3 tools)

**vurl_ip_bypass** -- Generate IP spoofing headers (X-Forwarded-For, X-Real-IP, etc.) to bypass IP-based access controls.

**vurl_auth_bypass** -- Generate authentication bypass headers for proxy-level auth bypass attacks.

**vurl_identity** -- Identity protocol attacks: OIDC front-channel logout injection, session fixation, Entra ID cross-tenant sync.

Actions: `oidc_logout`, `categories`

---

### AI/LLM Security (6 tools)

**vurl_mcp_rce** -- Generate AI agent exploitation payloads for MCP RCE, Langflow, CrewAI, AutoGPT, LangChain, OpenAI Assistants, Claude MCP, Gemini, Dify, Flowise.

**vurl_langflow** -- Langflow-specific exploitation payloads for CVE-2025-3248 and other RCE vectors.

**vurl_tool_payloads** -- Generate exploitation payloads for specific AI agent tool types: http_request, file_read, file_write, code_exec, shell_exec, database, browser.

**vurl_llm_poison** -- Generate LLM context poisoning payloads for RAG injection, system prompt extraction, jailbreaks, and tool manipulation.

Actions: `generate`, `types`

**vurl_shadow_ai** -- Generate shadow AI prompt injection payloads for exploiting AI agents via reflected content.

Actions: `generate`, `json`, `html`, `categories`, `context`, `detection`

**vurl_ai_gateway** -- AI Gateway bypass payloads for Cloudflare AI, AWS Bedrock, Azure Content Safety.

Actions: `detect`, `bypass`, `encode`, `gateways`

---

### Payload Generation (2 tools)

**vurl_payload** -- Generate protocol payloads for SSRF exploitation: Redis, Memcached, SMTP, FastCGI, MySQL, PostgreSQL, LDAP, DNS, Gopher, CoAP (IoT), MQTT (IoT), Dict, GraphQL, gRPC-Web, XXE.

**vurl_csd** -- Client-Side Desync (CSD) attack payloads: browser-powered desync attacks.

Actions: `generate`, `triggers`

---

### Edge Runtime and Server-Specific (4 tools)

**vurl_edge** -- Generate edge runtime attack payloads for Vercel Edge, Cloudflare Workers, Deno Deploy, Netlify Edge, AWS Lambda@Edge.

Actions: `generate`, `middleware`, `signatures`, `runtimes`, `categories`

**vurl_fluentbit** -- Generate Fluent Bit CVE-2025-12970 exploitation payloads. Targets 15+ billion deployments.

Actions: `generate`, `large_payload`, `ssrf_endpoints`, `manifests`, `detection`, `categories`

**vurl_sharepoint** -- Generate SharePoint CVE-2025-53770 exploitation payloads.

Actions: `generate`, `cve`, `high_risk`, `categories`

**vurl_rust_panic** -- Generate Rust panic DoS/RCE payloads targeting validation panics in Safe Rust wrappers around Unsafe C libraries. Based on CVE-2026-21895.

Actions: `generate`, `deepsurf`, `templates`, `endpoints`, `categories`, `targets`, `large`, `cves`, `microservices`

---

### DNS and Network (2 tools)

**vurl_rebind** -- DNS rebinding timing analysis for TOCTOU exploitation.

Actions: `analyze`, `payloads`, `timing`

**vurl_rebind_v2** -- DNS rebinding v2 with advanced async timing analysis for TOCTOU exploitation.

Actions: `analyze`, `timing`, `payloads`, `detect`, `resolve`

---

### Vector Database and RAG (1 tool)

**vurl_vectordb** -- Vector database injection payloads for Pinecone, Milvus, Weaviate, Qdrant, Chroma.

Actions: `detect`, `exploit`, `databases`

---

### Race Detection (1 tool)

**vurl_race** -- Race condition and parser differential detection: fire URLs at multiple endpoints and detect differences.

Actions: `quick`, `types`

---

### Asset Management (1 tool)

**assets** -- Asset inventory: unified recon database for SubFlow, XMass, vmap results.

CRUD: `list`, `get`, `create`, `update`, `delete`, `ports`, `events`. Intelligence: `stats`, `coverage`, `cluster_jarm`, `cluster_favicon`. Ingest: `ingest_subflow`, `ingest_xmass`, `ingest_vmap`. Pipeline: `crawl_seeds`.

---

### Mobile Security (1 tool)

**mobile** -- Mobile app security analysis: static analysis (APK/IPA), device management, dynamic instrumentation (Frida), proxy setup.

Actions: `toolchain`, `devices`, `device_info`, `emulator_start`, `emulator_list`, `analyze_apk`, `analyze_ipa`, `decompile`, `decode`, `manifest`, `network_config`, `binary_info`, `scan_secrets`, `apps`, `app_info`, `install`, `uninstall`, `launch`, `stop`, `clear_data`, `pull_apk`, `frida_ps`, `frida_apps`, `frida_spawn`, `frida_attach`, `ssl_bypass`, `root_bypass`, `objection_ssl`, `objection_env`, `objection_classes`, `objection_methods`, `proxy_setup`, `proxy_clear`, `proxy_check`, `push_ca`, `check_cleartext`, `ios_proxy_instructions`, `shared_prefs`, `read_shared_pref`, `databases`, `dump_database`, `app_files`, `pull_storage`, `logcat`, `crash_detect`, `syslog`, `crashes`, `raw_shell`, `forward`, `reverse`

---

### Collaboration (1 tool)

**collab** -- Real-time collaboration between hunters. Share your Hugin session with a teammate (Pro license required).

Actions: `share`, `join`, `status`, `leave`, `publish`

---

### Extensions and Tooling (3 tools)

**extensions** -- Lua extension management: load, unload, enable, disable extensions and test hooks.

Actions: `list`, `get`, `load`, `unload`, `enable`, `disable`, `reload`, `stats`, `test_hook`

**hugin_tools_registry** -- External tools registry: list registered tools, check health, execute tool commands.

**hugin_files** -- Manage payload file library: add, remove, list, preview, search files and manage tags.

---

### Campaigns and Automation (1 tool)

**hugin_campaigns** -- Manage automation campaigns: create, configure, start/stop multi-step intruder attacks with payload sets.

---

### Enterprise (4 tools)

**rbac** -- Role-Based Access Control: manage users, roles, permissions, and audit logging for multi-user environments.

Actions: `list_users`, `get_user`, `create_user`, `update_user`, `delete_user`, `list_roles`, `create_role`, `delete_role`, `check_permission`, `audit_log`, `log_action`

**invisible_proxy** -- Invisible (transparent) proxy mode: configure and manage transparent proxy interception without client-side proxy settings.

Actions: `status`, `configure`, `enable`, `disable`, `generate_rules`, `connections`, `dns_config`

**hugin_http3** -- HTTP/3 (QUIC) proxy configuration and analysis.

Actions: `status`, `configure`, `probe`, `alt_svc_scan`, `fingerprint`, `stats`

**hugin_spnego** -- SPNEGO/NTLM enterprise authentication: configure credentials, detect auth challenges, generate tokens.

Actions: `status`, `configure`, `list_credentials`, `delete_credential`, `detect`, `generate_type1`, `parse_challenge`, `generate_type3`, `sessions`, `test`

---

## MCP Resources

The MCP server also exposes resources for reading configuration and state:

- `hugin://config` -- Current proxy configuration
- `hugin://scope` -- Active scope patterns
- `hugin://stats` -- Proxy statistics (flow count, hosts, etc.)

## Usage Tips

- Start broad: use `list_flows` and `search_flows` to understand captured traffic before diving deep.
- Use `get_flow` with short ID prefixes (e.g., `get_flow(id: "a3f2")`) instead of copying full UUIDs.
- For WAF-protected targets: use `browser(action: "launch")` first, then `browser(action: "exec_js")` with `fetch()` to make requests through the browser's real TLS fingerprint.
- Chain tools together: `vurl_js_endpoints` to find endpoints, then `vurl_endpointer` to profile them, then `intruder` or `sqli` to test.
- Use `pipeline` to run all scanners in sequence on a host with one call.
- Track OOB callbacks with `vurl_oastify` -- it correlates callbacks to specific payloads automatically.
