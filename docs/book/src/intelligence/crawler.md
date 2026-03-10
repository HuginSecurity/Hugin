# Crawler

The crawler discovers URLs, forms, and API endpoints on a target by following links and analyzing page content. Every request it makes is captured as an `HttpFlow` and stored in the current project, making the results available for the scanner, Nerve, and other tools. It does not require you to browse the application manually.

## Crawl Strategies

Two traversal strategies are available:

**BFS (breadth-first)** -- explores all links at the current depth before going deeper. Produces more uniform coverage and terminates earlier when the depth limit is reached. This is the default.

**DFS (depth-first)** -- follows links as deep as they go before backtracking. Useful for reaching deeply nested pages that BFS might not reach within the page limit.

Both strategies use the same URL frontier for deduplication.

## Scope Control

The scope controls which URLs the crawler follows. Scope evaluation has this priority order:

1. **Exclude patterns** -- regex matched against the full URL. If any exclude pattern matches, the URL is out of scope regardless of other settings.
2. **Include patterns** -- regex matched against the full URL. If any include pattern matches, the URL is in scope.
3. **Seed host matching** -- the crawl stays on the same host as the seed URL(s). Subdomains can be included or excluded independently.

By default, `include_subdomains` is `true`, so a seed of `example.com` also follows `api.example.com` and `static.example.com`. Set it to `false` for strict host-only scope.

## Configuration

```json
{
  "max_depth": 3,
  "max_pages": 1000,
  "concurrency": 10,
  "delay_ms": 100,
  "per_host_delay_ms": 1000,
  "timeout_secs": 30,
  "respect_robots": true,
  "parse_sitemap": true,
  "submit_forms": false,
  "js_analysis": true,
  "enable_passive": false,
  "capture_redirects": true,
  "probe_methods": false,
  "strategy": "bfs"
}
```

- `max_depth` -- Maximum link depth from seed URLs. Default is 3.
- `max_pages` -- Maximum total pages to visit. The crawl stops when this is reached.
- `concurrency` -- Number of parallel worker tasks. The per-host rate limiter (`per_host_delay_ms`) operates independently from the global `delay_ms`.
- `respect_robots` -- Fetch and honor `robots.txt` for the target host before crawling.
- `parse_sitemap` -- Fetch and enqueue `sitemap.xml` at crawl start to seed the frontier.
- `submit_forms` -- Fill and submit detected forms with synthetic values (see Form Detection below).
- `js_analysis` -- Parse JavaScript for embedded URLs and API endpoints (see JavaScript Analysis below).
- `enable_passive` -- Fetch historical URLs from Wayback Machine, CommonCrawl, and AlienVault OTX before crawling.
- `capture_redirects` -- Record each redirect hop as a separate flow. Useful for spotting open redirects.
- `probe_methods` -- Issue HEAD and OPTIONS requests per URL in addition to GET. Detects allowed-method misconfigurations.
- `custom_headers` -- Headers added to every request (e.g. `Authorization`, session cookies).
- `proxy` -- Route all crawler traffic through an HTTP proxy (e.g. `http://127.0.0.1:8080`).

## JavaScript Analysis

When `js_analysis` is enabled, the crawler parses all JavaScript encountered -- both inline `<script>` blocks and external script files. It extracts:

- Relative and absolute URLs referenced in JS code
- API endpoint paths constructed in JS (string literals, template literals)
- External script source URLs

Discovered URLs are added to the frontier and crawled if they are in scope. This is a static analysis pass, not dynamic execution -- it finds strings that look like URLs without running the code.

The crawler also extracts inline event handlers from HTML attributes (`onclick`, `onsubmit`, `onchange`, etc.) and logs them for manual review.

## Form Detection and Submission

The crawler parses all HTML forms it encounters, including:

- Form action URL and method (GET/POST)
- Input field names, types, and default values
- Hidden fields

When `submit_forms` is enabled, the form submitter fills detected forms with synthetic values appropriate to the field type (text fields get `test`, email fields get `test@test.com`, number fields get `1`, etc.) and submits them. The resulting request/response is captured as a flow.

Form submission respects the crawl scope -- forms that would submit to out-of-scope URLs are not submitted.

## Headless Browser Mode

When the `headless` feature is compiled and `headless_config` is set, the crawler spawns a headless Chromium browser for pages that require JavaScript to render:

- Executes JavaScript and waits for network idle before extracting links
- Captures XHR and fetch requests made during page load as flows
- Detects SPA routing and captures client-side navigation

Headless mode is slower and more resource-intensive than the default HTTP mode. Use it selectively for SPAs where static analysis misses significant application surface.

## Passive URL Sources

When `enable_passive` is true, the crawler fetches historical URLs from:

- Wayback Machine (web.archive.org)
- CommonCrawl
- AlienVault OTX

These URLs are added to the frontier before crawling begins. This can surface paths that are no longer linked from the live application but still exist on the server.

## Trap Detection

The crawler includes a trap detector that prevents infinite loops caused by:

- Calendars with infinite next/previous month links
- Pagination that generates unbounded page numbers
- Session tokens embedded in URLs that create unique URLs per visit
- Dynamic content that generates unique paths on every request

When a URL pattern is identified as a trap, the crawler skips it and logs the detection.

## robots.txt

When `respect_robots` is enabled, the crawler fetches `robots.txt` from the seed host's root and applies all `Disallow` and `Allow` rules for the configured user agent. Disallowed paths are not added to the frontier and are not visited.

To crawl disallowed paths (for authorized penetration testing), set `respect_robots: false`.

## User Agent Rotation

The crawler supports a pool of user agent strings that rotate across requests. By default it uses a fixed user agent string. Configure a pool of agents to rotate through on each request, which can help bypass simple bot-detection rules.

## Exported Flows

Every URL the crawler visits is stored as an `HttpFlow` in the project. The flow includes the request (method, URL, headers, body), the response (status, headers, body up to 512 KB), timing data, and metadata tags identifying the source as the crawler.

These flows appear in the proxy logger immediately and are available for scanning, Nerve analysis, and export.

## MCP Tool: `crawler`

**Actions:**

- `start` -- Begin a crawl with seed URLs and configuration.
- `stop` -- Stop the current crawl.
- `pause` / `resume` -- Pause and resume a crawl in progress.
- `status` -- Get crawl progress (pages visited, queue depth, URLs found).
- `urls` -- Get the list of discovered URLs.
- `export` -- Export discovered URLs in `json`, `csv`, or `txt` format.

There is also a standalone `vurl_crawler` MCP tool that uses the Vurl HTTP engine and works without the Hugin proxy running. It supports multiple routing modes (direct, Mullvad, Hugin proxy, custom proxy) and has its own session management with `start`, `stop`, `pause`, `resume`, `status`, `urls`, `export`, `list`, and `delete` actions.
