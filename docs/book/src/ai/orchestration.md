# Orchestration Engine

The orchestration engine enables LLM agents to autonomously explore and test web applications using Hugin's full toolkit. It lives in `hugin-service/src/orchestration/` and is accessed via the Assistant tab's Agent and Auto sub-tabs.

## Architecture

```
MCP (Claude/external) ──┐
UI (Assistant tab) ──────┤──▸ OrchestrationEngine ──▸ LlmProvider ──▸ OpenRouter/Ollama
Context menus ───────────┘          │
                                    ├── ToolBridge (41 tools)
                                    ├── ResponseCache (5-tier)
                                    ├── TaskPool (concurrent execution)
                                    ├── ExplorationMemory (cross-session dedup)
                                    ├── ModelRouter (per-task routing)
                                    └── AutoMode (4-phase autonomous assessment)
```

## Agent Tools (41)

The orchestrator exposes 41 tools to LLM agents via `ToolBridge`. Tool calls are dispatched directly to Rust service methods — no HTTP or MCP serialization overhead.

### Security Tools (14)

`repeater_send`, `repeater_raw_send`, `flows_get`, `flows_list`, `flows_search`, `scope_list`, `findings_list`, `findings_create`, `scanner_check`, `smart_decode_detect`, `paramhunter_discover`, `intruder_list`, `intruder_results`

### Browser Tools (5)

`browser_navigate`, `browser_exec_js`, `browser_source`, `browser_screenshot`, `browser_stop`

The orchestrator shares the same `BrowserMap` as the rest of the application. When the LLM calls `browser_navigate`, it opens Chrome through Hugin's proxy — all traffic is captured as flows. The browser session persists across tool calls.

### UI Automation Tools (6)

`ui_navigate`, `ui_list_components`, `ui_click`, `ui_set_input`, `ui_read_state`, `ui_screenshot`

### Read-Only Tools (15)

`sitemap_tree`, `cookie_jar_list`, `comparer_diff`, `sequencer_analyze`, `intercept_rules`, `projects_list`, `events_recent`, `environment_get`, `campaigns_list`, `workflows_list`, `extensions_list`, `scanner_scans`, `websocket_list`, `scheduler_jobs`, `assets_list`

### Modify Tools (1)

`modify_request` — NL-driven HTTP request modification

## Approval Policy

Tools that send external HTTP requests (like `repeater_send`, `browser_navigate`) have `sends_external: true`. Under the `SendOnly` approval policy, these pause for human approval before executing.

## Auto-Mode

4-phase autonomous assessment:

1. **Recon** — crawl, sitemap analysis, technology fingerprinting
2. **Passive Scan** — header analysis, cookie security, information disclosure
3. **Active Probe** — targeted scanner checks based on passive findings
4. **Deep Dive** — manual-style exploration of high-value targets

Each phase has a human checkpoint before proceeding.

## 5-Tier Cache

1. Exact hash (SHA-256)
2. Semantic similarity (256-dim embeddings, adaptive threshold)
3. Pattern cache (security header structures)
4. Disk persistent (`~/.config/hugin/cache/`)
5. Provider-level (Anthropic cache_control, OpenAI prefix caching)
