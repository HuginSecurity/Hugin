# Built-in Scanner

Hugin ships a native vulnerability scanner compiled directly into the binary. Unlike Synaps (WASM modules) or vurl (MCP offensive tools), the built-in scanner requires no plugins, no compilation, and no Pro license. It analyzes HTTP flows captured by the proxy, running 41 active checks and 35 passive checks against each target.

Active checks send modified requests with crafted payloads to detect vulnerabilities. Passive checks analyze existing request/response pairs without generating additional traffic.

## Scanner Features

- **OOB detection** -- Oastify integration for out-of-band callbacks (DNS, HTTP) across SQLi, CMDi, XSS, SSRF, XXE, LDAP, JWT, and Deserialization checks
- **Baseline-relative timing** -- adaptive threshold calibration eliminates false positives on slow servers; the scanner measures normal response time before issuing time-based payloads
- **Multi-request verification** -- checks like Server-Side Prototype Pollution use a pollute-then-verify pattern across separate requests to confirm impact
- **WAF detection** -- automatically identifies WAF block pages and skips them to avoid flooding findings with false positives
- **CWE auto-classification** -- every finding is tagged with the relevant CWE identifier
- **Configurable concurrency** -- control parallelism, rate limiting, and retry-on-timeout behavior
- **Scan profiles** -- five built-in profiles to balance speed and coverage

### Scan Profiles

| Profile | Description |
|---------|-------------|
| **Quick** | Fast surface scan. Runs a subset of checks with minimal payloads. Good for initial reconnaissance. |
| **Normal** | Balanced coverage. Runs all checks with standard payload sets. Default profile. |
| **Thorough** | Maximum coverage. Extended payload sets, longer timing windows, deeper fuzzing. Slower but comprehensive. |
| **PassiveOnly** | No active traffic. Runs all 35 passive checks against existing flows. |
| **AuditOnly** | Active checks only, no passive analysis. Useful when passive findings have already been reviewed. |

## Active Checks (41)

### Injection

**SQL Injection** -- CWE-89
Error-based, time-based blind, boolean-based blind, and out-of-band (via Oastify DNS/HTTP callbacks). Tests across insertion points in query parameters, headers, cookies, and body fields.

**Cross-Site Scripting (Reflected)** -- CWE-79
Reflected XSS with context-aware payloads for HTML body, attribute, JavaScript, URL, and CSS contexts. Template injection variants for client-side frameworks.

**Command Injection** -- CWE-78
Unix and Windows command separators, IFS bypass techniques, and OOB verification via Oastify. Tests both inline execution and blind injection with time delays.

**Server-Side Template Injection** -- CWE-1336
Engine-specific payloads for Jinja2, Twig, Pug, EJS, and Go templates. Detects template evaluation through arithmetic expressions and function calls.

**NoSQL Injection** -- CWE-943
MongoDB-style operator injection (`$ne`, `$where`), boolean-based and time-based detection. Tests JSON body parameters and query string injection points.

**LDAP Injection** -- CWE-90
Filter manipulation, authentication bypass via wildcard injection, and OOB detection through LDAP callback payloads.

**XPath Injection** -- CWE-643
Boolean-based extraction, union queries, and function abuse (`string-length`, `substring`). Targets XML-backed authentication and search endpoints.

**Expression Language Injection** -- CWE-917
SpEL, OGNL, and MVEL expression evaluation. Detects code execution through runtime expression interpreters in Java frameworks (Spring, Struts).

**Email Header Injection** -- CWE-93
CC/BCC injection, Reply-To manipulation, and From header spoofing via CRLF sequences in email-related parameters.

**XML Injection** -- CWE-91
Element injection, attribute injection, CDATA breakout, and SOAP body manipulation. Tests XML parsers for improper input handling.

**Server-Side JavaScript Injection** -- CWE-94
Detection of `eval()` and `Function()` sinks in server-side JavaScript (Node.js). Time-based and error-based confirmation.

**HTTP Response Splitting / Header Injection** -- CWE-113
CRLF injection in response headers. Tests for header injection leading to response splitting, cache poisoning, or XSS via injected headers.

### Web Application

**SSRF** -- CWE-918
Internal IP ranges (127.0.0.1, 169.254.169.254, 10.x, 172.16.x, fd00::), cloud metadata endpoints (AWS IMDSv1/v2, GCP, Azure), DNS rebinding, and OOB confirmation via Oastify.

**XXE** -- CWE-611
File read via `file://`, SOAP-based XXE, SVG upload vectors, XInclude injection, and OOB exfiltration through external DTDs. Tests XML content types and multipart uploads.

**HTTP Request Smuggling** -- CWE-444
CL.TE, TE.CL, TE.TE obfuscation, and H2.CL downgrade attacks. Differential response analysis confirms desync between front-end and back-end servers.

**Insecure Deserialization** -- CWE-502
Gadget chains for Java (Commons Collections, Spring), PHP (`unserialize`), Python (pickle), .NET (BinaryFormatter, ObjectStateFormatter), Ruby (Marshal), Fastjson, Node.js, and JNDI lookup triggers. OOB verification for blind deserialization.

**JWT Attacks** -- CWE-347
Algorithm `none` bypass, RSA/HMAC confusion (RS256 to HS256), KID path traversal and SQL injection, weak secret brute-force, and JWK header embedding.

**OAuth/OIDC** -- CWE-863
`redirect_uri` bypass (open redirect, path traversal, subdomain takeover), missing `state` parameter, PKCE downgrade, and token leakage via referrer.

**CSRF** -- CWE-352
Token removal, content-type switching (JSON to form), and HTTP method override to bypass CSRF protections.

**Session Fixation** -- CWE-384
URL-based session tokens, cookie fixation before authentication, path-scoped cookies, and `__Host-`/`__Secure-` prefix bypass.

**Race Conditions** -- CWE-362
Concurrent request replay, single-packet attack (multiple requests in one TCP segment), TOCTOU exploitation, and double-spend verification.

### Client-Side and Infrastructure

**Client-Side Prototype Pollution** -- CWE-1321
Injects `__proto__`, `constructor.prototype`, and `Object.prototype` payloads via query parameters and JSON bodies. Verifies pollution through reflected property access.

**Server-Side Prototype Pollution** -- CWE-1321
Multi-request verification: sends a pollution payload, then a separate request to confirm the prototype chain was modified on the server. Detects status code changes, new headers, and body mutations.

**CORS Active Probing** -- CWE-942
Tests null origin, full reflection, prefix/suffix matching (e.g., `evil-example.com`, `example.com.evil.com`), and special character injection in the Origin header.

**Cache Poisoning** -- CWE-349
Unkeyed header injection (X-Forwarded-Host, X-Original-URL), unkeyed query parameter injection, and parameter cloaking via semicolons and duplicate keys.

**Web Cache Deception** -- CWE-525
Path extension appending (`.css`, `.js`, `.png`), path delimiter confusion (`/profile%00.css`), and encoding tricks to make authenticated responses cacheable.

**BOLA/IDOR** -- CWE-639
Sequential numeric ID iteration, UUID enumeration, base64-encoded ID tampering, Relay/GraphQL global ID decoding, and privilege escalation by substituting resource identifiers.

**File Upload** -- CWE-434
Extension bypass (double extension, null byte, case variation), content-type mismatch, polyglot files (GIF89a + PHP), web.config/`.htaccess` upload, and path traversal in filenames.

**HTTP/2 Attacks** -- CWE-444
Pseudo-header injection (`:authority`, `:path`), CRLF in HTTP/2 header values, and method confusion between HTTP/2 and HTTP/1.1 backends.

**Client-Side Desync** -- CWE-444
Browser-compatible CL.TE desync attacks that poison the browser's connection pool, enabling same-origin request hijacking.

**Open Redirect** -- CWE-601
Absolute URL redirect (`//evil.com`), protocol-relative URLs, `javascript:` and `data:` URI schemes, and backslash/encoded variations.

**Host Header Injection** -- CWE-644
Password reset poisoning, cache poisoning via Host header, and protocol downgrade through X-Forwarded-Proto manipulation.

**HTTP Method Override** -- CWE-650
Override headers (`X-HTTP-Method-Override`, `X-Method-Override`), query parameter override (`_method=PUT`), and body parameter override.

**HTTP Parameter Pollution** -- CWE-235
Framework-specific duplicate parameter handling: PHP (last wins), ASP.NET (comma-join), Express (array), Java (first wins). Tests for authorization and filter bypass.

**Mass Assignment** -- CWE-915
Flat parameter injection (`is_admin=true`), nested object injection, Rails bracket notation (`user[role]`), Django/Spring/Mongoose-specific patterns, and `$set` operator injection.

**HTTP Method Testing** -- CWE-650
TRACE method reflection, WebDAV methods (PROPFIND, MOVE, COPY), and verb tampering to bypass access controls.

**Stored/Blind XSS** -- CWE-79
Payload injection with OOB verification for blind XSS, mXSS through HTML sanitizer bypass, DOM clobbering, Angular template injection, and Oastify callback confirmation.

**GraphQL Info Disclosure** -- CWE-200
Introspection query enablement, query batching abuse, alias-based DoS amplification, and Automatic Persisted Query (APQ) cache bypass.

**GraphQL Authorization Bypass** -- CWE-862
IDOR through GraphQL node IDs, field-level authorization gaps, nested relationship traversal, and mutation-level access control bypass.

**Keycloak Misconfiguration** -- CWE-287
Admin console XSS, JWT signature forgery via JWKS endpoint manipulation, open user registration, and reverse proxy bypass to access admin endpoints.

**WebSocket Security** -- CWE-1385
Cross-Site WebSocket Hijacking (CSWSH via Origin validation), message injection through parameter tampering, token replay across sessions, and authentication persistence after logout.

## Passive Checks (35)

Passive checks run automatically on every flow captured by the proxy. They never send additional requests.

**Security Headers** -- Missing or misconfigured Content-Security-Policy, Strict-Transport-Security, X-Frame-Options, and X-Content-Type-Options headers.

**Sensitive Data Detection** -- API keys, AWS access keys, private keys (RSA, EC, Ed25519), JWTs, OAuth tokens, and other credentials in response bodies.

**Cookie Security** -- Missing `HttpOnly`, `Secure`, or `SameSite` attributes on session and authentication cookies.

**Information Disclosure** -- Server version headers, stack traces, debug pages, and verbose error messages revealing internal details.

**CORS Misconfiguration** -- Wildcard `Access-Control-Allow-Origin`, full origin reflection, and `null` origin acceptance.

**OAuth Flow Analysis** -- Implicit flow usage (token in URL fragment), missing `state` parameter, and absent PKCE challenge.

**Deserialization Content-Type Detection** -- Responses with content types indicating serialized objects (Java serialization, PHP serialization, .NET ViewState).

**Mixed Content Detection** -- HTTPS pages loading resources over HTTP (scripts, stylesheets, images, iframes).

**Open Redirect (Passive)** -- URL parameters reflecting in `Location` headers or `meta` refresh tags without validation.

**Cacheable Sensitive Response** -- Responses containing sensitive data (authentication tokens, PII) served with permissive cache headers.

**Directory Listing Detection** -- Server-generated directory index pages (Apache, Nginx, IIS).

**Clickjacking** -- Missing `X-Frame-Options` and `frame-ancestors` CSP directive on authenticated pages.

**CSP Detailed Analysis** -- Identifies `unsafe-inline`, `unsafe-eval`, `data:`, `blob:`, wildcard sources, and other CSP weaknesses.

**Content-Type Mismatch** -- Response body content not matching the declared Content-Type header (e.g., HTML served as `text/plain`).

**Subresource Integrity** -- External scripts and stylesheets loaded without `integrity` attributes.

**Referrer Policy** -- Missing or overly permissive Referrer-Policy header leaking URLs to third parties.

**Permissions Policy** -- Missing Permissions-Policy (formerly Feature-Policy) header for sensitive APIs (camera, microphone, geolocation).

**HSTS Detailed** -- Short `max-age` values, missing `includeSubDomains`, and absent `preload` directive.

**Private IP Disclosure** -- Internal IP addresses (RFC 1918, link-local, loopback) exposed in response headers or bodies.

**Email Disclosure** -- Email addresses in response bodies that may reveal internal contacts or organizational structure.

**Credit Card Detection** -- Credit card number patterns with Luhn algorithm validation to reduce false positives.

**Stack Trace Disclosure** -- Language-specific stack traces for Java, .NET, PHP, Python, Ruby, Node.js, Go, and Rust.

**Software Version Disclosure** -- Version strings for web servers, frameworks, libraries, and CMS platforms.

**Internal Path Disclosure** -- Absolute file system paths (Unix and Windows) in responses revealing server directory structure.

**ViewState Analysis** -- .NET ViewState decoding, MAC validation detection, and sensitive data exposure in unencrypted ViewState.

**Password Autocomplete** -- Password fields without `autocomplete="off"` or `autocomplete="new-password"`.

**Cleartext Password Submission** -- Login forms submitting credentials over unencrypted HTTP.

**Session Token in URL** -- Session identifiers passed as URL query parameters instead of cookies.

**Sensitive Data in URL** -- Passwords, tokens, and API keys present in URL query strings.

**Cross-Domain Referer Leak** -- Sensitive URLs leaked to third-party domains via the Referer header.

**Input Reflection** -- User input reflected verbatim in responses without encoding, indicating potential injection points.

**Outdated JavaScript Libraries** -- Version detection for jQuery, AngularJS, Bootstrap, Lodash, Moment.js, and other libraries with known CVEs.

**DOM XSS Source/Sink Analysis** -- JavaScript source-to-sink data flow analysis identifying `location.hash`, `document.referrer`, `postMessage`, and other DOM XSS vectors.

**HTML Comment Mining** -- Extracts TODO comments, credentials, internal URLs, API keys, and developer notes from HTML comments.

**Form Security** -- Forms with cleartext `action` URLs, missing CSRF tokens, or autocomplete enabled on sensitive fields.

## How to Use

### UI

1. Open the **Scanner** tab from the sidebar.
2. Select one or more flows from the History view to scan.
3. Choose a scan profile (Quick, Normal, Thorough, PassiveOnly, or AuditOnly).
4. Start the scan. Progress is shown in real time with check names and status.
5. Findings appear in the **Findings** tab as they are discovered.

### MCP Tool

The `scanner` MCP tool provides full control over the scanning engine.

```
scanner action:"scan" flow_id:"<uuid>" profile:"normal"
```

Key actions:

- `scan` -- scan a specific flow or list of flows
- `status` -- check scan progress
- `stop` -- cancel a running scan
- `findings` -- list findings from completed scans
- `profiles` -- list available scan profiles

Example -- scan a flow with the Thorough profile:

```
scanner action:"scan" flow_id:"abc123" profile:"thorough"
```

Example -- run passive checks only:

```
scanner action:"scan" flow_id:"abc123" profile:"passive_only"
```

### Oastify Integration

For checks that support out-of-band detection, Oastify must be running. Start the OOB listener before scanning:

```
oastify action:"start"
```

The scanner automatically generates Oastify payloads for relevant checks (SQLi, CMDi, XSS, SSRF, XXE, LDAP, JWT, Deserialization) and polls for callbacks after payload delivery.

## Architecture Note

The built-in scanner is one of three scanning systems in Hugin:

- **Built-in Scanner** (this chapter) -- 41 active + 35 passive checks, compiled into the binary, no license required
- **[Synaps](./synaps.md)** -- WASM-based community modules, sandboxed runtime, Pro license
- **[vurl](./vurl.md)** -- 56 MCP offensive tools for manual testing, Pro license

All three systems write findings to the same store, viewable in the unified [Findings](../core/findings.md) tab.
