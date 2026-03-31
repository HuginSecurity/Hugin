# Taint Analysis

Runtime DOM XSS detection via browser-based taint tracking. Traces data flow from attacker-controlled sources to dangerous sinks using CDP JavaScript injection.

## How It Works

1. **Inject** — `TAINT_INSTRUMENTATION_JS` is injected into the page via `execute_js()`. It hooks source reads and sink writes using `Object.defineProperty`, `Proxy`, and function patching.
2. **Navigate** — the target URL loads with hooks active. Every time tainted data reaches a sink, a flow record is created.
3. **Collect** — `TAINT_COLLECTOR_JS` harvests `window.__huginTaintFlows` from the page and same-origin iframes.
4. **Classify** — each flow gets a severity based on sink type and a CWE category.
5. **Report** — confirmed flows are auto-created as `FindingRecord` entries in the store, deduped by URL + sink type.

## Source Types (7)

| Source | What it tracks |
|--------|---------------|
| `url_param` | `location.search` parameters |
| `url_hash` | `location.hash` fragment |
| `referrer` | `document.referrer` |
| `cookie` | `document.cookie` reads |
| `window_name` | `window.name` |
| `postmessage` | `event.data` from `message` events |
| `storage` | `localStorage.getItem()` / `sessionStorage.getItem()` |

## Sink Types (14)

| Sink | Risk | Severity |
|------|------|----------|
| `innerHTML` | DOM XSS | High |
| `outerHTML` | DOM XSS | High |
| `document.write` | DOM XSS | High |
| `eval` | Code execution | High |
| `Function` constructor | Code execution | High |
| `setTimeout(string)` | Code execution | High |
| `setInterval(string)` | Code execution | High |
| `insertAdjacentHTML` | DOM XSS | High |
| `jQuery.html` | DOM XSS | High |
| `dynamic_script` | Script injection | High |
| `location.assign` | Open redirect | Medium |
| `location.replace` | Open redirect | Medium |
| `location.href` | Open redirect | Medium |
| `script.src` | Script injection | Medium |

## Usage

### MCP Tool: `hugin_taint`

**All-in-one analysis:**
```
action: "analyze"
url: "https://target.com/page?q=test"
wait_ms: 5000        # optional, default 3000
interact: true       # optional, triggers hashchange/popstate events
```

**Manual workflow:**
```
action: "inject"     # inject hooks into active browser
# ... browse manually ...
action: "collect"    # harvest flows
```

### Orchestrator (LLM Agent)

The LLM chains browser + JS tools:

1. `browser_navigate(url: "https://target.com")` — open in proxied Chrome
2. `browser_exec_js(script: "<instrumentation JS>")` — inject hooks
3. `browser_exec_js(script: "document.querySelector('input').value = '<img src=x>'")` — simulate input
4. `browser_exec_js(script: "<collector JS>")` — harvest taint flows
5. `findings_create(...)` — record confirmed DOM XSS

### Auto-Findings

When `analyze` finds source→sink flows, it automatically creates findings:

- **Title**: `DOM XSS: url_param → innerHTML` (example)
- **Severity**: High (DOM XSS sinks) or Medium (redirect sinks)
- **CWE**: CWE-79 (DOM XSS), CWE-601 (Open Redirect), CWE-829 (Cross-Origin)
- **Evidence**: source value, sink element, JavaScript call stack
- **Dedup key**: SHA-256 of URL + sink type (prevents duplicate findings)

## Implementation

- Types: `hugin-shared/src/taint.rs` — `TaintFlow`, `TaintAnalysisResult`, severity/category classifiers
- JS: `TAINT_INSTRUMENTATION_JS` (~350 lines) + `TAINT_COLLECTOR_JS` in same file
- MCP tool: `hugin-mcp/src/tools/taint.rs` — `analyze`, `collect`, `inject` actions
- Orchestrator: browser tools in `hugin-service/src/orchestration/tools.rs`

## Resilience

- Every hook wrapped in `try/catch` — instrumentation never breaks the page
- Errors collected in `window.__huginTaintErrors` for debugging
- Idempotent — re-injection checks `window.__huginTaintInstrumented` flag
- Values truncated to 500 chars to prevent memory issues
- Works with CSP since injection is via CDP (not external script tag)
