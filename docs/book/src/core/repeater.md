# Repeater

Repeater lets you replay captured HTTP flows with arbitrary modifications. Change headers, body, method, URL, or redirect the request to a different server -- all without touching your browser.

## Workflow

1. Right-click any flow in the History view and choose "Send to Repeater", or use the MCP tool to send it programmatically.
2. The flow appears in the Repeater tab.
3. Edit headers, body, URL, or method in the editor panel.
4. Click Send. The response appears immediately alongside the original.
5. The exchange is recorded in repeater history and linked back to the original flow.

## Request Modifications

Every send operation accepts modifications:

- **headers_add** -- list of `[name, value]` pairs appended to the request
- **headers_remove** -- list of header names to strip (case-insensitive)
- **body** -- replaces the request body (UTF-8 string, raw bytes, or null to remove)
- **url** -- replaces the URL; can be absolute (`https://other.host/path`) or relative (`/api/v2/users`)
- **method** -- replaces the HTTP verb
- **target_override** -- redirects the TCP connection to a different `host:port` while keeping the original URL in the request line

When a URL is relative (no `://`), Hugin merges it with the scheme and authority of the original URL. This lets you test path variations without retyping the full URL.

## Browser-Routed Requests

Set `browser_port` to route the request through a running browser's `fetch()` API. This gives the request a real browser TLS fingerprint (JA3/JA4), bypassing WAF bot detection systems like Akamai BMP, DataDome, and Cloudflare Bot Management.

## Batch Send Modes

Send multiple requests in one operation with three concurrency modes:

- **sequential** (default) -- send one at a time, wait for each response
- **sync** -- fire all requests simultaneously from a synchronized barrier; useful for race condition testing
- **parallel** -- concurrent with a configurable worker limit

Batch results include per-request status codes, latency, response size, and the new flow ID for each replay. Aggregate timing statistics (min, max, mean, stddev) are included when sending more than one request.

## Raw Send

The `raw_send` action sends a single raw HTTP request via a direct socket connection with full TLS and SOCKS5 proxy support. Use `batch_raw_send` for parallel raw requests. The `repeat` action sends raw requests through the proxy so the flow is captured in history.

## Queue Management

The queue is persistent and backed by SQLite. If you close Hugin and reopen it, pending requests survive.

- **list** -- view all pending items with status (pending, in_progress, completed, failed)
- **cancel** -- remove a specific item before it is sent
- **clear** -- remove all pending items

History is capped at 1,000 entries. Older entries are pruned automatically.

## History

Every completed send is recorded. History entries include:

- Original flow ID and new flow ID (the replayed pair)
- Method, URL, HTTP status code
- Latency in milliseconds
- Timestamp

## MCP Tool

**`repeater`** -- HTTP request repeater.

Actions: `send`, `history`, `queue`, `queue_status`, `raw_send`, `batch_raw_send`, `repeat`, `repeat_flow`, `batch_send`, `list_queue`, `cancel`, `clear_queue`, `proxy_status`, `compare`

## API

```
POST /api/repeater/queue              {"flow_id": "uuid", "modifications": {...}}
POST /api/repeater/send               {"id": "uuid-of-queued-item"}
POST /api/repeater/batch              {"items": [...], "mode": "sync"}
GET  /api/repeater/history?limit=50
```
