# Intercept

The Intercept view lets you observe, inspect, and modify HTTP traffic in real time as it flows through the proxy. It operates in three states controlled by two toolbar toggles.

## Operating Modes

### Intercept OFF (default)

Full passthrough. Nothing is captured or displayed in the intercept view. Traffic flows through the proxy silently and is recorded to history as normal.

Header pill: **Intercept off** (green).

### Observe Mode (Intercept ON, Hold OFF)

Traffic flows through the proxy without interruption -- the browser never stalls. Requests appear in the intercept view as a live feed (last 500, ring buffer). You can scroll through them and inspect details, but you cannot modify or block them since they have already been sent.

Use this when you want to watch traffic without interfering: mapping the application, understanding flows, looking for interesting endpoints.

The **Clear** button wipes the captured list.

Header pill: **Intercept** (green).

### Hold Mode (Intercept ON, Hold ON)

Blocking mode. Every request that matches the intercept scope (and is not auto-handled by an Intercept Rule) is held in a queue. The browser stalls until you release the request.

Four action buttons appear:

- **Forward** -- release the selected request as-is (or after editing in the detail pane)
- **Forward All** -- release all queued requests at once
- **Drop** -- discard the selected request; the browser gets an error
- **Drop All** -- discard all queued requests

Click a request in the list to select it, inspect its full HTTP content in the detail pane below, edit headers or body, then click Forward to send the modified version.

Header pill: **Hold** (red).

**Keyboard shortcuts:** `Ctrl+;` forwards the selected request, `Ctrl+D` drops it.

## Scope

Intercept only captures requests to hosts matching the configured scope patterns. Empty scope means capture everything. Scope is configured in the Scopes view and shared with the proxy.

## Capture Options

Two checkboxes in the toolbar control what gets captured beyond standard requests:

- **Responses** -- also intercept HTTP responses (hold mode only)
- **WebSockets** -- also intercept WebSocket messages

When response interception is enabled, responses are held separately and can be individually forwarded, modified, or dropped.

## Intercept Rules

The gear icon opens the Intercept Rules panel. Rules run before the observe/hold decision and can automatically forward, drop, modify, or flag requests without manual intervention. See [Intercept Rules](./intercept-rules.md) for the full reference.

## MCP Tool

**`intercept`** -- MITM intercept control.

Actions:

- `status` -- current state (enabled, hold, pending counts, recording status)
- `toggle` -- set enabled/hold/intercept_responses/recording
- `recording` -- toggle flow recording on or off
- `list` -- list pending requests (hold mode)
- `get` -- get a specific pending request
- `forward` -- forward a held request
- `drop` -- drop a held request
- `forward_all` -- forward all held requests
- `drop_all` -- drop all held requests
- `responses` -- list pending responses
- `response_get` -- get a specific pending response
- `response_forward` -- forward a held response
- `response_drop` -- drop a held response
- `responses_forward_all` -- forward all held responses
- `responses_drop_all` -- drop all held responses

## API

Direct REST access for scripting:

```
GET  /api/intercept/status
POST /api/intercept/toggle              {"enabled": true, "hold": false}
GET  /api/intercept/pending
GET  /api/intercept/captured
POST /api/intercept/captured/clear
POST /api/intercept/{id}/forward
POST /api/intercept/{id}/drop
POST /api/intercept/forward-all
POST /api/intercept/drop-all
```
