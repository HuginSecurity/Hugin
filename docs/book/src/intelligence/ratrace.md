# RatRace -- Race Condition Testing

RatRace tests endpoints for race conditions by sending multiple concurrent requests with precisely synchronized timing. The goal is to compress the request window so that concurrent state checks at the server overlap, bypassing rate limits, double-spend protections, and other concurrency guards.

## Sync Engines

Five engines control how requests are synchronized before release.

**Barrier** -- All worker tasks build their complete request and wait at a tokio barrier. When all workers have arrived, the barrier releases them simultaneously. Works well for targets over local or low-latency links. This is the default engine for most tests.

**LastByte** -- Holds back the final bytes of each request until all connections are ready, then releases the last bytes at once. All workers establish a TCP connection and write the request body up to `hold_percentage` of the payload, then wait. On release, the held bytes are written simultaneously. Three sync methods:

- `TcpNagle` -- disables Nagle's algorithm so each byte is sent immediately. Effective over low-RTT links.
- `H2Multiplex` -- multiplexes all requests over a single HTTP/2 connection. The server receives all DATA frames in one TCP segment.
- `SinglePacket` -- forces all DATA frames into a single TCP packet using socket-level control. Tightest synchronization available.

**SinglePacket** -- Opens one HTTP/2 connection and sends all requests as concurrent streams. Each stream is a separate concurrent request from the server's perspective but all arrive in the same TCP segment.

**GraphqlBatch** -- For GraphQL endpoints, packs multiple query operations into a single batched request body. The server processes them concurrently if batching is enabled.

**Auto** -- Probes the target for H2 support and selects `SinglePacket` if available, otherwise falls back to `LastByte` with `TcpNagle`.

## Race Enhancements

Enhancements modify the base configuration to probe for different race condition variants:

- **Jitter** -- Adds random delay (0 to `jitter_max_ms`) to each worker before release. Tests endpoints that detect uniform timing.
- **Warmup** -- Issues N requests before the actual race to warm connection pools and JIT paths. Reduces cold-start variance.
- **Token rotate** -- Uses a different session token per request. Detects per-token rate limits that fail to properly isolate state.
- **Method mix** -- Varies the HTTP method (GET/POST/HEAD) across concurrent requests. Triggers server-side dispatch divergence.
- **Header race** -- Adds `X-Race-Request: 1` and similar headers. Some caching layers or middleware handle these differently.
- **Protocol hop** -- Sends some requests over HTTP/1.1 and others over HTTP/2. Useful when the load balancer routes them to different backend instances.
- **Lag induction** -- Deliberately delays one request by N milliseconds within the race window. Probes TOCTOU windows of specific widths.
- **Lock timeout** -- Tests with configurable lock wait thresholds to find timeout-based race windows.
- **Fuzz** -- Mutates parameter values across concurrent requests to test input-dependent state handling.

## Endpoint Discovery

RatRace automatically classifies endpoints into 17 race-prone categories:

- `payment` -- Payment processing, checkout, charge, invoice
- `auth` -- Login, signin, 2FA/MFA, OTP verification
- `referral` -- Promo codes, coupons, vouchers, rewards, claims
- `inventory` -- Stock, cart, reservations, bookings, allocations
- `file` -- Uploads, imports, attachments
- `webhook` -- Webhook callbacks, notifications
- `social` -- Votes, likes, follows, favorites, reviews
- `api` -- Transfers, withdrawals, deposits, balance mutations
- `ghost_write` -- Comments, posts, messages, replies
- `limit_reset` -- Rate limit resets, quota updates
- `state_transition` -- Approve/reject, enable/disable, activate/cancel
- `subscription_downgrade` -- Plan changes, tier upgrades/downgrades
- `mfa_bypass` -- MFA verification endpoints
- `double_dip_webhook` -- Payment webhooks (Stripe, PayPal IPN)
- `password_reset_race` -- Password reset, forgot-password flows
- `cdn_cache` -- Static assets, cache purge endpoints
- `ssrf_file_swap` -- URL fetch, proxy, preview, screenshot endpoints

Each category has associated HTTP methods and a base confidence score. The `discover` action scans a list of URLs and returns race-prone endpoints ranked by confidence.

## Static Code Scanner

RatRace includes a static code scanner that finds race condition patterns in source code. It supports 6 languages:

- **JavaScript/TypeScript** -- Non-atomic read-modify-write, missing locks, async race patterns
- **Python** -- Thread-unsafe globals, missing GIL-aware locks, async race conditions
- **Go** -- Goroutine data races, missing mutex, channel misuse
- **Ruby** -- Thread-unsafe class variables, missing synchronize blocks
- **PHP** -- File-based locking issues, session race conditions
- **Java/Kotlin/Scala** -- Missing synchronized blocks, non-atomic compound operations, ConcurrentModificationException patterns

## Race Signals

Every race result is classified into one of seven signals:

- `CleanCatch` -- Multiple requests produced divergent responses with clear indication of a logic flaw (e.g., two 200 responses on a one-time-use action).
- `DirtyCatch` -- Divergent responses observed but with noise. Likely real but needs confirmation.
- `SilentSuccess` -- All requests succeeded but no observable divergence.
- `LogicalRace` -- Divergent response codes or bodies that indicate a logic conflict (e.g., one 200 and one 409).
- `FalsePositive` -- Divergence explained by normal server behavior (CDN variance, load balancer routing).
- `RateLimited` -- The server detected the burst and throttled or blocked requests.
- `NoSignal` -- All requests returned identical responses.

## Detection

RatRace computes a `ResponseFingerprint` for each response: status code, body length, structural hash of JSON bodies, key header values, and latency. Timing statistics (min, max, mean, median, stddev, p95, p99) are recorded per window.

Additional detection includes:

- **Response divergence** -- Different body hashes within a race window
- **Status code divergence** -- Mixed status codes (e.g., 200 + 409 + 429)
- **Body length divergence** -- Significant body size differences
- **Database signals** -- Race-related error patterns (deadlock, duplicate key, serialization failure) in response bodies
- **State verification** -- Before/after state comparison to detect double-spend or count violations

## Protocol-Specific Tests

- **GraphQL Subscription Race** -- Tests concurrent subscription establishment for duplicate event delivery.
- **OIDC Logout Race** -- Tests concurrent logout + session-use to detect post-logout session persistence.
- **WebSocket Race** -- Tests concurrent WebSocket upgrade requests for connection state races.
- **Cache Race** -- Tests concurrent requests to detect cache poisoning windows during miss-to-hit transitions.

## Distributed Testing

- **Microservice mode** -- Sends concurrent requests to multiple different endpoints simultaneously. Detects cross-service races where operations on different microservices share backend state.
- **Orchestrate mode** -- Coordinates race tests across multiple distributed agents for geographically dispersed testing.

## Reporting

RatRace generates reports in three formats:

- `json` -- Machine-readable session data with all timing statistics and evidence
- `csv` -- Tabular results for spreadsheet analysis
- `html` -- Visual report with signal classification and timing charts

## MCP Tool: `ratrace`

**Core actions:**

- `test` -- Full race test with complete configuration
- `detect` -- Auto-detection mode (probes for race susceptibility)
- `quick` -- Quick test with minimal configuration (URL, method, body, request count)
- `limit` -- Rate limit bypass test
- `batch` -- YAML batch configuration for multiple tests
- `sessions` -- List all race test sessions
- `session` -- Get a specific session by ID
- `cancel` -- Cancel an in-progress session
- `result` -- Get detailed results for a session

**Discovery:**

- `discover` -- Find race-prone endpoints from a URL list
- `param_hunt` -- Hunt for parameters within a race category
- `endpoints` -- List all known race endpoint patterns
- `patterns` -- List all 17 endpoint classification patterns

**Static scanning:**

- `scan` -- Scan source code for race condition patterns
- `scan_findings` -- Get scan findings

**Protocol-specific:**

- `ws` -- WebSocket race test
- `microservice` -- Cross-service race test
- `cache_race` -- Cache poisoning race test
- `state_fuzz` -- State transition fuzzing
- `orchestrate` -- Multi-node distributed orchestration
- `graphql_subscription` -- GraphQL subscription race test
- `oidc_logout` -- OIDC logout race test

**Engine configuration:**

- `engine_config` -- Configure engine parameters
- `engine_timing` -- Get timing analysis for a session
- `engine_graphql` -- Configure GraphQL batch engine

**Enhancements:**

- `enhance_fuzz` -- Configure parameter fuzzing
- `enhance_lock` -- Configure lock timeout testing
- `enhance_protocol` -- Configure protocol hopping
- `enhance_lag` -- Configure lag induction

**Detection:**

- `detect_db` -- Detect database race signals in responses
- `detect_node` -- Detect node divergence across distributed tests
- `detect_thresholds` -- Configure detection thresholds

**Advanced:**

- `blind` -- OOB (blind) race testing
- `multisession` -- IDOR-style multi-session races
- `report` / `report_custom` -- Generate reports
