# Active Scanner

The active scanner is the built-in engine that replays captured flows with attack payloads and analyzes responses for vulnerabilities. It operates on `HttpFlow` records -- requests you have already captured through the proxy. It does not crawl or discover endpoints on its own; it tests what you have already seen.

## The 43 Active Checks

All checks are registered at startup. Each has a string ID used for filtering.

**Injection**

- `active-sqli` -- SQL Injection. Error-based (15+ SQL error signatures), time-based (5s delay via SLEEP/pg_sleep/WAITFOR), differential (baseline comparison).
- `active-xss` -- Cross-Site Scripting. Reflected payload patterns across HTML body, attribute, JS, and URL contexts.
- `active-stored-xss` -- Stored/Blind XSS. Injects callback payloads that fire on later page views. OOB confirmation.
- `active-cmdi` -- OS Command Injection. Error strings, time delays, OOB callbacks.
- `active-ssti` -- Server-Side Template Injection. Math evaluation (`{{7*7}}` -> 49) across Jinja2, Twig, Freemarker, Velocity, Pebble, Mako, Smarty engines.
- `active-ssjs` -- Server-Side JavaScript Injection. Node.js `eval()` and `Function()` injection patterns.
- `active-el-injection` -- Expression Language Injection. Java EL (`${7*7}`), Spring SpEL, OGNL injection.
- `active-xpath-injection` -- XPath Injection. Boolean-based and error-based payloads.
- `active-ldap-injection` -- LDAP Injection. Special characters, wildcard injection, OOB callbacks via JNDI.
- `active-nosql` -- NoSQL Injection. MongoDB operator injection (`$gt`, `$ne`, `$regex`), JavaScript injection in `$where`.
- `active-xml-injection` -- XML Injection. Entity expansion, attribute injection, CDATA breakout.
- `active-xxe` -- XML External Entity. OOB exfiltration, file content patterns (`root:`, `[boot loader]`), parameter entities.
- `active-email-injection` -- Email Header Injection. CRLF injection in email headers, BCC/CC injection.
- `active-header-injection` -- HTTP Response Splitting. CRLF injection in response headers.

**Authentication and Authorization**

- `active-csrf` -- Cross-Site Request Forgery. Missing token detection, origin header bypass, SameSite cookie analysis.
- `active-jwt` -- JWT Attacks. `none` algorithm, HS/RS confusion, JKU/X5U hijack, key brute-force.
- `active-bola` -- BOLA/IDOR. Numeric ID enumeration, UUID swapping, path-based object reference testing.
- `active-session-fixation` -- Session Fixation. Pre-authentication session token reuse detection.
- `active-oauth` -- OAuth/OIDC Active Testing. Redirect URI manipulation, state parameter bypass, PKCE downgrade, token leakage.
- `active-keycloak` -- Keycloak IAM Misconfiguration. Realm enumeration, token endpoint exposure, well-known endpoint checks.
- `active-mass-assignment` -- Mass Assignment. Admin flag injection (`isAdmin`, `role`, `privilege`), hidden field discovery.
- `active-graphql-authz` -- GraphQL Authorization Bypass. Field-level access control testing, introspection-based permission probing.

**Request Smuggling and Desync**

- `active-http-smuggling` -- HTTP Request Smuggling. CL.TE, TE.CL, TE.TE timing differential detection.
- `active-csd` -- Client-Side Desync. Browser-based request smuggling gadgets, connection reuse exploitation.
- `active-http2` -- HTTP/2 Specific Attacks. H2.CL desync, HPACK bomb, pseudo-header injection, stream multiplexing abuse.

**Cache and Redirect**

- `active-cache-poisoning` -- Web Cache Poisoning. Unkeyed header injection, X-Forwarded-Host, X-Original-URL, poisoned response caching verification.
- `active-cache-deception` -- Web Cache Deception. Path confusion attacks, extension appending, delimiter injection to cache sensitive responses.
- `active-open-redirect` -- Open Redirect. Protocol-relative URLs, double-URL encoding, backslash tricks, domain confusion.

**Server-Side**

- `active-ssrf` -- Server-Side Request Forgery. OOB callback domain, cloud metadata URLs (169.254.169.254), DNS rebinding, protocol smuggling.
- `active-deserialization` -- Insecure Deserialization. Java ysoserial patterns, .NET formatters, PHP `unserialize()`, Python pickle. OOB callbacks.
- `active-race-condition` -- Race Condition. Concurrent request divergence detection.
- `active-file-upload` -- File Upload Vulnerability. Extension bypass, content-type mismatch, polyglot files, double extension, null byte injection.

**Prototype Pollution**

- `active-prototype-pollution` -- Client-Side Prototype Pollution. `__proto__` and `constructor.prototype` injection via query, body, and JSON.
- `active-ss-prototype-pollution` -- Server-Side Prototype Pollution. Status code differential, JSON reflection, delayed effect detection.

**Protocol and Method**

- `active-websocket` -- WebSocket Security. Upgrade bypass, CSWSH (cross-site WebSocket hijacking), message injection, origin validation.
- `active-graphql` -- GraphQL Injection. Introspection leak, query batching, field suggestion leakage, alias-based DoS.
- `active-host-header` -- Host Header Injection. Password reset poisoning, web cache poisoning via Host, X-Forwarded-Host, absolute URL override.
- `active-method-override` -- HTTP Method Override. X-HTTP-Method-Override, X-Method-Override, _method parameter for access control bypass.
- `active-http-method` -- HTTP Method Testing. OPTIONS enumeration, PUT/DELETE access, TRACE XST.
- `active-hpp` -- HTTP Parameter Pollution. Duplicate parameter injection to bypass WAF rules and server-side validation.
- `active-cors` -- CORS Misconfiguration. Origin reflection, null origin, subdomain wildcard, credential inclusion with permissive origins.

## Insertion Points

The scanner automatically extracts injection points from each flow:

- `QueryParam` -- every `key=value` pair in the query string
- `BodyParam` -- form-urlencoded body fields
- `JsonField` -- all string values in JSON bodies, including nested paths (e.g. `json[data.user.id]`)
- `Header` -- `User-Agent`, `Referer`, `X-Forwarded-For`, `Authorization`, and custom `X-*` headers
- `Cookie` -- each individual cookie name-value pair
- `UrlPath` -- path segments
- `XmlElement` -- XML body (for XXE; the check replaces the full body)

For each extracted point, the scanner runs the `TransformEngine` to detect nested encodings. If a parameter value is Base64-encoded JSON, the payload is re-encoded through the same transform chain before injection. This happens transparently.

## Detection Modes

Each payload carries an `ExpectedBehavior` that determines how the response is analyzed:

**Error-based** -- looks for a specific pattern string in the response body. SQL error messages, stack traces, template math output.

**Time-based** -- triggers if response time meets or exceeds a threshold in milliseconds. The default time-based SQL injection threshold is 5000ms.

**Differential** -- compares the response against a cached baseline for the same insertion point. The first request to a given point becomes the baseline. Subsequent payloads are compared using `ResponseComparator`. A similarity score below 0.85 (85%) flags a finding.

**Out-of-band** -- sends a payload containing the configured callback domain and marks it as triggered immediately. Actual OOB confirmation happens asynchronously through Oastify or an external collaborator.

## Scan Profiles

Five predefined profiles configure the scanner for different assessment needs:

**Quick** -- Fewer payloads (max 2 per insertion point), no time-based checks, no OOB, 10 concurrent requests, 15s timeout. For fast initial assessment.

**Normal** -- Default payload set, time-based enabled, OOB enabled, 5 concurrent requests, 30s timeout. Standard behavior.

**Thorough** -- All payloads (unlimited per point), extended timeouts (60s), retry on timeout, 3 concurrent requests. For deep assessment.

**PassiveOnly** -- No active probing. Only passive checks on observed traffic.

**AuditOnly** -- Active checks but non-invasive only. Skips SQLi DELETE payloads, file upload, and other destructive checks. Time-based and OOB enabled.

## Scan Configuration

```json
{
  "profile": "normal",
  "request_delay_ms": 100,
  "max_concurrency": 5,
  "timeout_secs": 30,
  "enabled_checks": [],
  "enabled_locations": [],
  "follow_redirects": false,
  "verify_ssl": true,
  "callback_domain": null,
  "max_payloads_per_point": 1000
}
```

- `profile` -- One of `quick`, `normal`, `thorough`, `passive_only`, `audit_only`. Overrides individual settings.
- `enabled_checks` -- List of check IDs to run. Empty means all 43 checks.
- `enabled_locations` -- List of insertion location types to test. Empty means all applicable.
- `callback_domain` -- Domain for OOB detection (SSRF, XXE, LDAP, JWT, deserialization, stored XSS). Without this, OOB checks fall back to error-based detection only.
- `max_payloads_per_point` -- Caps the number of payloads per insertion point. Default is 1000; set to 0 for no limit.
- `request_delay_ms` -- Per-host delay between requests. The rate limiter tracks the last request time per host independently.
- `max_concurrency` -- Maximum parallel in-flight requests. Backed by a tokio semaphore.

## Passive Checks (29 checks)

Passive checks analyze captured traffic without sending any requests. They run automatically on every flow.

**Security Header Checks**

- `security-headers` -- Missing Security Headers. Checks for Strict-Transport-Security, Content-Security-Policy, X-Content-Type-Options, X-Frame-Options, Permissions-Policy, Referrer-Policy.
- `hsts-detailed` -- HSTS Detailed Analysis. Max-age too short, missing includeSubDomains, missing preload, HSTS on HTTP (useless).
- `csp-detailed` -- CSP Detailed Analysis. unsafe-inline, unsafe-eval, wildcard sources, missing directives, overly permissive domains, base-uri missing.
- `clickjacking` -- Clickjacking Protection Missing. No X-Frame-Options or frame-ancestors CSP directive on HTML responses.
- `referrer-policy` -- Referrer-Policy Check. Missing or overly permissive policy that leaks URLs to third parties.
- `permissions-policy` -- Permissions-Policy Check. Missing policy or overly permissive feature grants (camera, microphone, geolocation).

**Information Disclosure**

- `sensitive-data` -- Sensitive Data Exposure. API keys, tokens, credentials in response bodies.
- `information-disclosure` -- Information Disclosure. Server version headers, powered-by headers, debug information.
- `private-ip-disclosure` -- Private IP Address Disclosure. RFC1918 addresses (10.x, 172.16-31.x, 192.168.x) in responses.
- `email-disclosure` -- Email Address Disclosure. Email addresses in response bodies.
- `credit-card-disclosure` -- Credit Card Number Disclosure. PAN patterns with Luhn validation.
- `stack-trace-disclosure` -- Stack Trace Disclosure. Java, .NET, Python, PHP, Ruby, Node.js stack traces.
- `software-version-disclosure` -- Software Version Disclosure. Library and framework versions in responses.
- `internal-path-disclosure` -- Internal Path Disclosure. Server filesystem paths (`/var/www`, `C:\inetpub`, `/home/`).

**Cookie and Session**

- `cookie-security` -- Insecure Cookie Configuration. Missing Secure, HttpOnly, SameSite attributes.
- `passive-session-token-url` -- Session Token in URL. Session IDs passed via query parameters.
- `passive-cleartext-password` -- Cleartext Password Submission. Passwords sent over HTTP.
- `passive-password-autocomplete` -- Password Autocomplete. Password fields without `autocomplete="off"`.

**Content and Response**

- `cors-misconfiguration` -- CORS Misconfiguration. Overly permissive Access-Control-Allow-Origin, credential inclusion.
- `mixed-content` -- Mixed Content. HTTPS pages loading HTTP resources (scripts, stylesheets, images).
- `content-type-mismatch` -- Content-Type Mismatch. Response body type does not match Content-Type header.
- `sri-missing` -- Missing Subresource Integrity. External scripts and stylesheets without SRI hashes.
- `cacheable-response` -- Cacheable Sensitive Response. Sensitive endpoints returning cache-friendly headers.
- `directory-listing` -- Directory Listing. Apache/Nginx/IIS directory indexes enabled.
- `open-redirect` -- Open Redirect. Redirect responses pointing to user-controlled URLs.

**Application Logic**

- `oauth-flow` -- OAuth/OIDC Flow Analysis. Missing state parameter, token in fragment, insecure redirect URIs.
- `deserialization-content-type` -- Dangerous Deserialization Content Types. Java serialized objects, XML with DOCTYPE, YAML load, pickle in request/response.
- `passive-viewstate` -- ViewState Analysis. Unencrypted or unsigned ASP.NET ViewState.

**Reflection and Leakage**

- `passive-input-reflection` -- Input Reflection. Request parameters reflected unencoded in response body.
- `passive-referer-leak` -- Cross-Domain Referer Leak. Sensitive URLs leaked to third-party domains via Referer header.
- `passive-sensitive-url` -- Sensitive Data in URL. Passwords, tokens, API keys in query parameters.

## MCP Tool: `scanner`

**Actions:**

- `start` -- Start a scan with `flow_ids` and optional config.
- `status` -- Current scan progress.
- `checks` -- List all available active checks.
- `findings` -- List findings from the current or a specific scan.
- `cancel` / `pause` / `resume` -- Control an in-progress scan.
- `clear` -- Clear all findings.
- `list_scans` -- Scan history.
- `get_scan` / `delete_scan` -- Individual scan management.
- `get_finding` / `delete_finding` / `update_finding` -- Finding management.
- `audit_items` -- Group findings by check ID for a specific scan.
- `create_finding` -- Manually create a finding.
- `add_finding_flow` / `remove_finding_flow` / `list_finding_flows` -- Associate flows with findings.
- `add_finding_tag` / `remove_finding_tag` / `list_finding_tags` -- Tag findings.

## Findings

Each finding records:

- The check that produced it
- The insertion point name and location
- The payload that triggered the detection
- The raw HTTP request with the payload injected
- The raw HTTP response
- Detection evidence (pattern matched, time delta, similarity score)
- Severity inherited from the check

Findings are stored per-flow and queryable by flow ID, check ID, severity, or tag. The probe request/response (the exact traffic that produced the finding) is stored separately and retrievable for inclusion in reports.

## OOB Callback Integration

When `callback_domain` is configured, checks that support OOB (SSRF, XXE, LDAP injection, JWT, deserialization, stored XSS) automatically include Oastify payloads. The scanner marks those findings as triggered immediately so they appear in the findings list. To confirm them, poll Oastify for the correlation ID embedded in the payload.

Checks without a `callback_domain` fall back to error-based detection only. OOB confirmation upgrades finding confidence to `Confirmed`.
