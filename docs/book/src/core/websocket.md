# WebSocket

Hugin captures, stores, and lets you inspect and manipulate WebSocket traffic bidirectionally. Every frame flowing through the proxy is recorded with direction, type, payload, and timestamp.

## How Capture Works

When the proxy receives an HTTP request with `Upgrade: websocket`, it hands the connection to the WebSocket manager. The original HTTP 101 handshake is recorded as a flow in the usual history. All subsequent frames are stored as WebSocket messages linked to that flow.

The proxy runs two concurrent relay tasks per connection: one for client-to-server frames, one for server-to-client frames. Each frame passes through a capture and optional interception step before being forwarded.

## Frame Types

All WebSocket frame opcodes are tracked:

- **Text** -- UTF-8 text payload, previewed in the UI
- **Binary** -- raw bytes, displayed as hex or byte count
- **Ping** -- keepalive sent by either side
- **Pong** -- keepalive reply
- **Close** -- connection termination frame with optional close code and reason

## Connection Lifecycle

When an upgrade completes, a WebSocket connection is registered:

- **id** -- UUID for the connection
- **flow_id** -- links to the original HTTP 101 flow
- **url** / **host** -- the WebSocket endpoint
- **message_count** -- incremented on each captured frame
- **state** -- Open, Closing, or Closed

Active connections are held in memory. When a connection closes, the database record is updated with the close timestamp.

## Message Storage

Each captured message is persisted with:

- Connection ID
- Direction (ClientToServer or ServerToClient)
- Opcode
- Payload bytes
- Timestamp

Messages are queryable by connection ID, direction, and time range.

## Real-time Streaming

The manager provides two broadcast channels for live updates:

- A message channel emitting each captured frame -- the UI and event stream receive frames in real time
- A connection channel emitting connect and disconnect events

Both channels are buffered (1,000 messages, 100 connections). Slow consumers miss old messages but never block the proxy.

The MCP tool supports streaming via `poll_messages` and `poll_connections` actions with `since_id` or `since` parameters for incremental polling.

## Manual Intercept

WebSocket intercept works like HTTP intercept. When enabled, each frame is held in a pending queue before being forwarded. You can:

- **Forward** -- release the frame unchanged
- **Forward Modified** -- change the opcode and/or payload before forwarding
- **Drop** -- discard the frame; neither side receives it

Each pending message shows direction, opcode, full payload, and timestamp. The UI previews the first 200 characters for text frames or a byte count for binary frames.

## Message Injection

Inject arbitrary frames into any active WebSocket connection from outside the browser:

```json
POST /api/websocket/{connection_id}/send
{
  "opcode": "Text",
  "payload": "{\"action\": \"subscribe\", \"channel\": \"admin\"}"
}
```

The payload is forwarded to the server as if it originated from the browser. Binary payloads are accepted as Base64-encoded bytes.

Injection only works while the connection is active. Sending to a closed connection returns a 404.

## Listing Connections and Messages

```
GET /api/websocket/connections
GET /api/websocket/connections/{id}
GET /api/websocket/connections/{id}/messages?direction=client_to_server&limit=100
```

Active connections show `closed_at: null`. Historical connections include the close timestamp.

## Use Cases

- **Protocol reverse engineering** -- capture and inspect the message format before crafting modified payloads
- **Privilege escalation** -- inject messages the UI never sends (admin actions, restricted channel subscriptions)
- **Message tampering** -- intercept a frame mid-flight and change amounts, IDs, or action names
- **Replay attacks** -- send stored messages to test idempotency or re-trigger server-side actions
- **State manipulation** -- drop specific frames to test how the application handles missing messages

## MCP Tool

**`websocket`** -- WebSocket traffic inspection and manipulation.

Actions:

- `connections` -- list all WebSocket connections
- `get` -- get a specific connection by ID
- `messages` -- list messages for a connection (filter by direction, limit)
- `send` -- inject a message into an active connection
- `delete` -- delete a connection record
- `poll_messages` -- stream new messages (with since_id or since for incremental polling)
- `poll_connections` -- stream new connection events
- `intercept_status` -- check if WebSocket intercept is enabled
- `intercept_toggle` -- enable or disable WebSocket intercept
- `intercept_list` -- list pending intercepted frames
- `intercept_get` -- get a specific pending frame
- `intercept_forward` -- forward a pending frame
- `intercept_drop` -- drop a pending frame
- `intercept_forward_all` -- forward all pending frames
- `intercept_drop_all` -- drop all pending frames
