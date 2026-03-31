# Discovery

Discovery is a content discovery tool that brute-forces directories, files, and backup paths on a target web server. It probes a wordlist of paths against a URL base, filters responses by status code, and presents results in a sortable table with live progress.

## Quick Start

1. Open the **Discovery** tab from the Plugins section in the sidebar.
2. Enter the target URL (e.g., `https://target.com/`).
3. Select a wordlist (Common is a good default).
4. Click **Start**.
5. Watch results populate in real time, sorted by status code.

## Configuration

### Target URL

The base URL to scan. Discovery appends each wordlist entry as a path segment. Trailing slashes are handled automatically.

### Wordlists

Four built-in wordlists are available:

- **Common (~130 paths)** -- a curated list covering admin panels, config files, backup paths, API endpoints, dotfiles (`.env`, `.git/config`, `.htaccess`), CMS paths (WordPress, Drupal), and common framework routes.
- **raft-large-dirs** -- directory-focused wordlist from the RAFT collection.
- **raft-large-files** -- file-focused wordlist from the RAFT collection.
- **dirsearch-default** -- the default wordlist from dirsearch.
- **Custom** -- paste your own wordlist, one path per line. Lines starting with `#` are treated as comments.

### Extensions

Comma-separated file extensions to append to each wordlist entry that does not already contain a dot. Default: `.php,.html,.js,.txt`.

For example, with extensions `.php,.html` and the wordlist entry `admin`, Discovery probes:
- `/admin`
- `/admin.php`
- `/admin.html`

Entries that already contain a dot (like `robots.txt`) are sent as-is without extension appending.

### Threads

Number of concurrent requests. Default: 10. Higher values increase speed but may trigger rate limiting or WAF blocks.

### Status Filter

Comma-separated HTTP status codes to include in results. Default: `200,301,302,403`. Responses with other status codes are discarded. Leave empty to capture all responses.

### Follow Redirects

When enabled, Discovery follows HTTP redirects (301, 302, 307, 308) and reports the final response. When disabled (default), redirects are reported as-is.

### Recursive Mode

When enabled, directories found during the scan (status 200 or 301 with a directory-like path) are added to the scan queue for recursive exploration. Maximum recursion depth is 3 levels, with a queue limit of 50 sub-scans to prevent runaway scanning.

## Results

Results appear in a sortable table with five columns:

- **Status** -- HTTP status code, color-coded (green for 2xx, yellow for 3xx, red for 4xx/5xx)
- **URL** -- the discovered path
- **Size** -- response body size
- **Content-Type** -- the response content type
- **Response Time** -- request latency in milliseconds

Click any column header to sort. Click again to reverse the sort direction.

## Progress

While running, the toolbar shows:

- Progress bar with percentage (completed / total requests)
- Request rate (req/s)
- Error count
- Pause / Resume and Stop buttons

## Export

Click the **Export CSV** button to download all results as a CSV file.

## MCP Tool

**`discover`** -- content discovery scanner.

Actions:

- `run` -- run a discovery scan. Parameters: `url` (target), `wordlist` (path list or built-in name), `extensions` (comma-separated), `threads`, `status_codes` (filter), `follow_redirects`, `recursive`, `timeout_ms`.
- `wildcard_check` -- probe for wildcard responses before scanning. Sends requests with random paths to detect servers that return 200 for everything.

The MCP tool runs standalone via `vurl_http` -- no proxy required. Results include status code, body size, content type, response time, word count, and line count for each discovered path.
