# REST API

The Hugin API server exposes a REST API for automation, scripting, and integration with external tools. Every feature accessible through the UI is also accessible via REST.

## Base URL and Default Port

```
http://127.0.0.1:8081/api/
```

The API port defaults to 8081 and is configurable via `--api-port` or the config file. An OpenAPI spec is served at `/openapi.json`.

## Authentication

When running locally (bound to 127.0.0.1), authentication is disabled by default. When exposed on a network interface (0.0.0.0), enable authentication. See [Authentication](authentication.md) for details.

## Endpoint Reference

All endpoints are prefixed with `/api/` unless otherwise noted.

### Infrastructure

```
GET  /api/health               Health check (no auth required)
GET  /api/update/check         Check for updates
GET  /api/ca.pem               Download CA certificate
GET  /api/proxy/status         Proxy status and statistics
```

### Flows

```
GET  /api/flows                List flows (query params: method, host, flagged, limit, offset)
POST /api/flows                Create/import a flow
GET  /api/flows/{id}           Get flow details
```

### Repeater

```
POST /api/repeater/send        Replay a request with modifications
POST /api/repeater/batch       Send batch requests
GET  /api/repeater/history     List replay history
GET  /api/repeater/queue       Queue status
POST /api/repeater/queue       Queue a request for later
GET  /api/repeater/queue/list  List queued requests
DELETE /api/repeater/queue     Clear the queue
DELETE /api/repeater/queue/{id} Cancel a queued request
POST /api/repeat               Send raw HTTP request
POST /api/flows/{id}/repeat    Replay a flow through the proxy
```

### Intercept

```
GET  /api/intercept                    Intercept status
POST /api/intercept                    Toggle intercept on/off
GET  /api/intercept/pending            List pending intercepted requests
GET  /api/intercept/pending/{id}       Get a pending request
POST /api/intercept/forward/{id}       Forward a request
POST /api/intercept/drop/{id}          Drop a request
POST /api/intercept/forward-all        Forward all pending
POST /api/intercept/drop-all           Drop all pending
GET  /api/intercept/captured           List captured intercepts
POST /api/intercept/captured/clear     Clear captured intercepts
```

**Response Intercept:**

```
GET  /api/intercept/responses                  List pending responses
GET  /api/intercept/responses/{id}             Get a pending response
POST /api/intercept/responses/forward/{id}     Forward a response
POST /api/intercept/responses/drop/{id}        Drop a response
POST /api/intercept/responses/forward-all      Forward all responses
POST /api/intercept/responses/drop-all         Drop all responses
```

### Rules

```
GET  /api/rules                List intercept rules
POST /api/rules                Create a rule
GET  /api/rules/{id}           Get rule details
PATCH /api/rules/{id}          Update a rule
DELETE /api/rules/{id}         Delete a rule
GET  /api/rules/groups         List rule groups
POST /api/rules/groups         Create a group
DELETE /api/rules/groups/{id}  Delete a group
```

### Scope

```
GET  /api/scope                Get scope configuration
POST /api/scope                Update scope
POST /api/scope/mode           Set scope mode
POST /api/scope/recording      Toggle recording
POST /api/scope/pattern        Add a scope pattern
DELETE /api/scope/pattern/{p}  Remove a scope pattern
```

### Scanner

```
POST /api/scanner/scan                     Start a scan
GET  /api/scanner/status                   Scan status
GET  /api/scanner/findings                 List findings
POST /api/scanner/findings                 Create a manual finding
DELETE /api/scanner/findings               Clear all findings
GET  /api/scanner/findings/{id}            Get finding details
DELETE /api/scanner/findings/{id}          Delete a finding
PATCH /api/scanner/findings/{id}           Update a finding
GET  /api/scanner/findings/{id}/flows      List flows for a finding
POST /api/scanner/findings/{id}/flows      Add flow to finding
DELETE /api/scanner/findings/{id}/flows/{flow_id}  Remove flow from finding
GET  /api/scanner/findings/{id}/tags       List finding tags
POST /api/scanner/findings/{id}/tags       Add tag to finding
DELETE /api/scanner/findings/{id}/tags/{tag}  Remove tag
GET  /api/scanner/findings/tags            List all tags across findings
POST /api/scanner/cancel                   Cancel active scan
POST /api/scanner/pause                    Pause scan
POST /api/scanner/resume                   Resume scan
GET  /api/scanner/checks                   List available checks
GET  /api/scanner/scans                    List scan history
GET  /api/scanner/scans/{id}               Get scan details
DELETE /api/scanner/scans/{id}             Delete a scan
POST /api/scanner/scans/{id}/cancel        Cancel specific scan
GET  /api/scanner/scans/{scan_id}/audit-items  Audit items for a scan
```

### Session

```
GET  /api/session/tokens              List tracked tokens
GET  /api/session/status              Session status
GET  /api/session/macro               List macros
POST /api/session/macro               Create a macro
GET  /api/session/macro/{id}          Get macro details
DELETE /api/session/macro/{id}        Delete a macro
POST /api/session/macro/{id}/execute  Execute a macro
POST /api/session/refresh             Refresh all sessions
```

### WebSocket

```
GET  /api/websocket/connections               List WS connections
GET  /api/websocket/connections/{id}          Get connection details
DELETE /api/websocket/connections/{id}        Close a connection
GET  /api/websocket/connections/{id}/messages Get messages
POST /api/websocket/connections/{id}/send     Send a message
GET  /api/websocket/stream/messages           SSE stream of WS messages
GET  /api/websocket/stream/connections        SSE stream of WS connections
```

**WebSocket Intercept:**

```
GET  /api/websocket/intercept                  Intercept status
POST /api/websocket/intercept                  Toggle WS intercept
GET  /api/websocket/intercept/pending          List pending
GET  /api/websocket/intercept/pending/{id}     Get pending message
POST /api/websocket/intercept/forward/{id}     Forward message
POST /api/websocket/intercept/drop/{id}        Drop message
POST /api/websocket/intercept/forward-all      Forward all
POST /api/websocket/intercept/drop-all         Drop all
```

### Projects

```
GET  /api/projects                             List projects
POST /api/projects                             Create project
GET  /api/projects/{id}                        Get project
PUT  /api/projects/{id}                        Update project
DELETE /api/projects/{id}                      Delete project
POST /api/projects/{id}/archive                Archive/unarchive
POST /api/projects/{id}/activate               Set as active project
GET  /api/projects/{id}/stats                  Project statistics
GET  /api/projects/{id}/scope                  Get project scope
PUT  /api/projects/{id}/scope                  Update project scope
POST /api/projects/{id}/assign-flows           Assign flows to project
POST /api/projects/{id}/scope/snapshots        Create scope snapshot
GET  /api/projects/{id}/scope/snapshots        List scope snapshots
GET  /api/projects/{id}/scope/snapshots/{sid}  Get snapshot
GET  /api/projects/{id}/export                 Export project
POST /api/projects/import                      Import project
POST /api/projects/deactivate                  Deactivate project context
```

**Legacy project API (file-based):**

```
POST /api/project/save     Save project to file
POST /api/project/load     Load project from file
GET  /api/project/recent   List recent projects
```

### Site Map

```
GET /api/hosts                          List all hosts
GET /api/sitemap/hosts                  List hosts with stats
GET /api/sitemap/hosts/{host}           Host details
GET /api/sitemap/hosts/{host}/paths     Paths for a host
GET /api/sitemap/hosts/{host}/flows     Flows for a host
GET /api/sitemap/paths/{host}/{path}    Path details
GET /api/sitemap/search                 Search URLs
GET /api/sitemap/stats                  Aggregated stats
GET /api/sitemap/tree                   Hierarchical tree
GET /api/sitemap/export                 Export site map
```

### Sequencer

```
GET  /api/sequencer/captures               List capture sessions
POST /api/sequencer/capture                Start a token capture
DELETE /api/sequencer/capture/{id}         Delete session
GET  /api/sequencer/capture/{id}/status    Session status
GET  /api/sequencer/capture/{id}/tokens    Get captured tokens
POST /api/sequencer/capture/{id}/stop      Stop capture
GET  /api/sequencer/capture/{id}/export    Export session
POST /api/sequencer/analyze                Analyze tokens
POST /api/sequencer/compare                Compare token sets
```

### Crawler

```
POST /api/crawler/start    Start a crawl
POST /api/crawler/stop     Stop crawl
POST /api/crawler/pause    Pause crawl
POST /api/crawler/resume   Resume crawl
GET  /api/crawler/status   Crawl status
GET  /api/crawler/urls     Discovered URLs
GET  /api/crawler/export   Export URLs
```

### Browser

```
POST /api/browser/launch       Launch a browser session
POST /api/browser/sweep        Full sweep (launch + crawl)
POST /api/browser/close        Close browser
GET  /api/browser/status       Browser status
POST /api/browser/navigate     Navigate to URL
POST /api/browser/exec_js      Execute JavaScript
POST /api/browser/screenshot   Take screenshot
GET  /api/browser/list         List active browser sessions
```

### Tunnel Mode

```
GET  /api/tunnel/domains        List tunnel domains
POST /api/tunnel/domains/add    Add a domain to tunnel mode
POST /api/tunnel/domains/remove Remove a domain from tunnel mode
```

### Oastify (Native OOB Server)

```
POST /api/oastify-native/start              Start OOB server
POST /api/oastify-native/stop               Stop OOB server
GET  /api/oastify-native/status             Server status
GET  /api/oastify-native/domain             Get callback domain
GET  /api/oastify-native/stats              Statistics
GET  /api/oastify-native/payloads           List payloads
POST /api/oastify-native/payloads           Generate a payload
POST /api/oastify-native/payloads/batch     Generate batch payloads
POST /api/oastify-native/payloads/all       All protocol payloads
GET  /api/oastify-native/payloads/{id}      Get payload details
GET  /api/oastify-native/interactions       List interactions
GET  /api/oastify-native/interactions/{id}  Get interaction
DELETE /api/oastify-native/interactions/{id} Delete interaction
POST /api/oastify-native/interactions/{id}/acknowledge  Acknowledge
POST /api/oastify-native/interactions/acknowledge-bulk  Bulk acknowledge
GET  /api/oastify-native/interactions/correlation/{id}  By correlation ID
GET  /api/oastify-native/poll               Poll for new interactions
GET  /api/oastify-native/stream             SSE stream of interactions
```

### Oastify (Remote Tracker)

```
GET  /api/oastify/status           Connection status
POST /api/oastify/connect          Connect to remote server
POST /api/oastify/disconnect       Disconnect
GET  /api/oastify/stats            Statistics
POST /api/oastify/generate         Generate tracked payload
GET  /api/oastify/payloads         List tracked payloads
GET  /api/oastify/interactions     List interactions
POST /api/oastify/sync             Sync callbacks
POST /api/oastify/register_batch   Register batch payloads
```

### Dashboard

```
GET /api/dashboard/events   Event log (query: limit, since_id)
GET /api/dashboard/tasks    Active tasks
GET /api/dashboard/stats    Aggregated statistics
```

### Events

```
GET    /api/events          List events (query: category, level, search, since, until)
DELETE /api/events          Clear all events
GET    /api/events/recent   Recent events from memory cache
GET    /api/events/stats    Event statistics
GET    /api/events/stream   SSE stream of events
DELETE /api/events/before   Delete events before timestamp
```

### RatRace

```
POST /api/ratrace/test                   Run race condition test
POST /api/ratrace/detect                 Detect race conditions
POST /api/ratrace/quick                  Quick race test
POST /api/ratrace/limit                  Rate limit test
POST /api/ratrace/batch                  Batch race test
GET  /api/ratrace/sessions               List sessions
GET  /api/ratrace/sessions/{id}          Get session
DELETE /api/ratrace/sessions/{id}        Delete session
POST /api/ratrace/sessions/{id}/cancel   Cancel session
GET  /api/ratrace/results/{id}           Get results
POST /api/ratrace/discover               Discover endpoints
POST /api/ratrace/param-hunt             Hunt for race-prone params
GET  /api/ratrace/endpoints              List discovered endpoints
POST /api/ratrace/scan                   Full race scan
GET  /api/ratrace/scan/findings          Scan findings
POST /api/ratrace/ws                     WebSocket race test
POST /api/ratrace/microservice           Microservice race test
POST /api/ratrace/cache-race             Cache race test
POST /api/ratrace/state-fuzz             State fuzzing
POST /api/ratrace/orchestrate            Multi-step orchestration
GET  /api/ratrace/report/{id}            Generate report
POST /api/ratrace/graphql-subscription   GraphQL subscription race
POST /api/ratrace/oidc-logout            OIDC logout race
```

### Intruder

```
GET  /api/intruder/attacks               List attacks
POST /api/intruder/attacks               Start an attack
GET  /api/intruder/attacks/{id}          Get attack details
DELETE /api/intruder/attacks/{id}        Delete attack
GET  /api/intruder/attacks/{id}/status   Attack status
POST /api/intruder/attacks/{id}/pause    Pause
POST /api/intruder/attacks/{id}/resume   Resume
POST /api/intruder/attacks/{id}/cancel   Cancel
GET  /api/intruder/attacks/{id}/results  Get results
GET  /api/intruder/generators            List payload generators
GET  /api/intruder/processors            List processing rules
```

### Comparer

```
POST /api/comparer/compare       Compare two responses
POST /api/comparer/blind-detect  Blind vulnerability detection
GET  /api/comparer/similarity    Calculate similarity score
```

### Extensions

```
GET  /api/extensions              List extensions
GET  /api/extensions/stats        Statistics
GET  /api/extensions/{id}         Get extension details
POST /api/extensions/{id}/load    Load
POST /api/extensions/{id}/unload  Unload
POST /api/extensions/{id}/enable  Enable
POST /api/extensions/{id}/disable Disable
POST /api/extensions/{id}/reload  Reload
POST /api/extensions/test-hook    Test a hook
```

### Settings

```
GET    /api/settings/upstream-proxy          Get upstream proxy config
POST   /api/settings/upstream-proxy          Set upstream proxy
DELETE /api/settings/upstream-proxy          Clear upstream proxy
GET    /api/settings/upstream-proxy/rules    List proxy rules
POST   /api/settings/upstream-proxy/rules    Add proxy rule
PUT    /api/settings/upstream-proxy/rules    Replace all rules
DELETE /api/settings/upstream-proxy/rules/{id}  Remove rule
POST   /api/settings/upstream-proxy/test     Test proxy connection
POST   /api/settings/upstream-proxy/preset   Apply preset (tor, burp, mullvad, etc.)
GET    /api/settings/http2                   Get HTTP/2 config
POST   /api/settings/http2                   Set HTTP/2 config
```

### Decoder

```
POST /api/decoder/encode              Encode data
POST /api/decoder/decode              Decode data
POST /api/decoder/chain               Chain encode/decode operations
POST /api/decoder/analyze             Analyze encoding layers
POST /api/decoder/reencode            Re-encode detected layers
POST /api/decoder/jwt/decode          Decode JWT
POST /api/decoder/jwt/forge           Forge JWT
POST /api/decoder/polyglot            Generate polyglot payloads
GET  /api/decoder/polyglot/contexts   List polyglot contexts
GET  /api/decoder/operations          List available operations
```

### Assets

```
GET  /api/assets                       List assets
POST /api/assets                       Create asset
GET  /api/assets/stats                 Statistics
GET  /api/assets/coverage              Pipeline coverage
POST /api/assets/ingest/subflow        Ingest SubFlow results
POST /api/assets/ingest/xmass          Ingest XMass results
POST /api/assets/ingest/vmap           Ingest vmap results
POST /api/assets/crawl-seeds           Generate crawl seeds from assets
GET  /api/assets/cluster/jarm/{hash}   Cluster by JARM hash
GET  /api/assets/cluster/favicon/{mmh3} Cluster by favicon hash
GET  /api/assets/{id}                  Get asset
DELETE /api/assets/{id}                Delete asset
PATCH /api/assets/{id}                 Update asset
GET  /api/assets/{id}/ports            List ports
GET  /api/assets/{id}/events           List events
```

### Intelligence

```
GET  /api/intelligence/params                   List discovered params
GET  /api/intelligence/params/search            Search params by name
GET  /api/intelligence/params/stats             Param statistics
GET  /api/intelligence/params/endpoint          Params for endpoint
GET  /api/intelligence/routes                   List routes
GET  /api/intelligence/routes/detail            Route details
GET  /api/intelligence/reflections              List reflected params
GET  /api/intelligence/reflections/candidates   XSS candidates
GET  /api/intelligence/reflections/stats        Reflection stats
GET  /api/intelligence/endpoints                List endpoints
GET  /api/intelligence/endpoints/detail         Endpoint details
GET  /api/intelligence/endpoints/stats          Endpoint stats
GET  /api/intelligence/rollups                  Security rollups
GET  /api/intelligence/rollups/summary          Rollup summary
GET  /api/intelligence/rollups/host/{host}      Host posture
POST /api/intelligence/analyze                  Analyze existing flows
```

### Nerve (Parameter Analysis)

```
POST /api/nerve/analyze             Analyze parameters
POST /api/nerve/analyze-flows       Analyze params from flows
GET  /api/nerve/categories          List categories
GET  /api/nerve/categories/{short}  Category details
GET  /api/nerve/stats               Statistics
```

### Collaboration (Pro)

```
POST /api/collab/share    Create shared session
POST /api/collab/join     Join via invite code
GET  /api/collab/status   Session status
POST /api/collab/leave    Leave session
POST /api/collab/publish  Publish annotation to partner
```

### Access Tokens

```
GET    /api/tokens          List tokens
POST   /api/tokens/create   Create token
DELETE /api/tokens/{token}  Revoke token
```

### License

```
GET  /api/license/status    License status
POST /api/license/account   Set account ID
```

### External Tools

```
GET  /api/tools                  List registered tools
GET  /api/tools/health           Health check all tools
GET  /api/tools/{name}           Get tool info
GET  /api/tools/{name}/health    Health check one tool
POST /api/tools/{name}/execute   Execute tool command
```

### Scheduler

```
GET    /api/scheduler/jobs              List jobs
POST   /api/scheduler/jobs              Create job
GET    /api/scheduler/jobs/{id}         Get job
PATCH  /api/scheduler/jobs/{id}         Update job
DELETE /api/scheduler/jobs/{id}         Delete job
POST   /api/scheduler/jobs/{id}/run     Trigger job
GET    /api/scheduler/jobs/{id}/runs    List run history
```

### GraphQL

```
GET  /graphql    GraphQL Playground (HTML)
POST /graphql    GraphQL query/mutation endpoint
```

### Real-time

```
GET /ws    WebSocket endpoint for real-time flow updates
```

## Response Format

All API responses return JSON. Successful responses typically include the data directly. Errors return HTTP status codes:

- `200` -- Success
- `400` -- Bad request (invalid parameters)
- `401` -- Unauthorized (auth required)
- `404` -- Not found
- `500` -- Internal server error

## CORS

The API enables permissive CORS (all origins, all methods, all headers) to allow browser-based tooling to connect from any domain.
