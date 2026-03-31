# Browser Automation

Hugin can control Chrome and Mullvad Browser programmatically through CDP (Chrome DevTools Protocol) and Marionette. Browser sessions are shared across the GUI, MCP, and orchestrator — all using the same `BrowserMap`.

## Access Paths

- **GUI** — click "Open Browser" in the toolbar
- **MCP** — `hugin_browser(action: "navigate", url: "...")` + 20 other actions
- **Orchestrator** — LLM agents call `browser_navigate`, `browser_exec_js`, etc.
- **API** — REST endpoints at `/api/browser/*`

All paths share the same browser instance. An orchestrator LLM can navigate to a page, and the user sees it live in the Chrome window.

## Orchestrator Browser Tools

Five tools available to LLM agents:

- **`browser_navigate`** — open browser and navigate to URL through Hugin's proxy. All traffic captured as flows. `sends_external: true` (triggers approval gate).
- **`browser_exec_js`** — execute JavaScript via CDP `Runtime.evaluate`. Supports async/await via auto-wrapping in async IIFE.
- **`browser_source`** — get current page HTML source (truncated to 500KB).
- **`browser_screenshot`** — capture page screenshot as base64 PNG.
- **`browser_stop`** — close browser session.

## Session Management

- Port-keyed: one browser per proxy port
- Auto-launch: `browser_navigate` creates a session if none exists
- Adopt-first: checks for existing Chrome on CDP port before launching new
- Crash recovery: auto-relaunch on `exec_js` if previous session exited

## MCP Browser Actions (20+)

Full browser control via MCP:

- Lifecycle: `browse`, `navigate`, `status`, `stop`
- Content: `exec_js`, `source`, `screenshot`
- Tabs: `new_tab`, `switch_tab`, `close_tab`, `list_tabs`
- Input simulation: `mouse_move`, `click`, `double_click`, `type_text`, `press_key`, `scroll`, `drag`, `hover`
- Elements: `find_element`, `click_element`, `type_into`
- Crawling: `crawl` (BFS with scope filtering)

## Chrome vs Mullvad

- **Chrome** (CDP) — full-featured, BoringSSL fingerprint blends with 65%+ of web traffic, anti-detect JS injected on launch
- **Mullvad Browser** (Marionette) — Firefox-based, Tor-compatible fingerprint, enhanced privacy defaults
