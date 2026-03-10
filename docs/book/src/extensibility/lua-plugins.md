# Lua Extensions

Lua extensions are scripts that run inside the Hugin proxy and react to traffic in real time. They are the only mechanism in Hugin that can modify requests and responses in flight -- changing a URL, injecting a header, rewriting a body, or dropping a request entirely before it reaches the server.

Extensions require a Pro license.

## Extension Structure

Each extension lives in its own directory under `~/.config/hugin/extensions/`:

```
~/.config/hugin/extensions/
  my-extension/
    extension.json    # Manifest (required)
    main.lua          # Entry point (required)
    lib/              # Optional helper modules
```

### Manifest (`extension.json`)

```json
{
  "id": "my-extension",
  "name": "My Extension",
  "version": "1.0.0",
  "author": "Your Name",
  "description": "What this extension does",
  "entry_point": "main.lua",
  "hooks": ["OnRequest", "OnResponse", "PassiveCheck"],
  "permissions": ["ReadFlows", "ModifyFlows"]
}
```

The manifest declares the extension's identity, which hooks it subscribes to, and what permissions it requires. Hugin auto-discovers manifests by scanning the extensions directory.

## Hooks

Extensions register interest in hooks via the manifest. Hugin calls the corresponding Lua function when the event fires. Six hook types are available:

### OnRequest

Called before each proxied request is forwarded to the target. Can modify or drop requests. Lua function: `on_request`.

```lua
function on_request(ctx)
    -- ctx.flow_id    (string) Unique flow identifier
    -- ctx.method     (string) HTTP method
    -- ctx.url        (string) Full URL
    -- ctx.headers    (table)  Key-value header pairs
    -- ctx.body       (string) Request body or nil

    -- Modify headers
    ctx.headers["X-Custom-Header"] = "injected"

    -- Return action:
    -- "continue"  - forward (with any modifications applied to ctx)
    -- "drop"      - drop the request entirely
    -- or return a table with modified fields:
    return {
        headers = ctx.headers,
        url = ctx.url,
        method = ctx.method,
        body = ctx.body
    }
end
```

### OnResponse

Called before each response is returned to the client. Can modify status, headers, body, or drop the response.

Lua function: `on_response`.

```lua
function on_response(ctx)
    -- ctx.flow_id    (string) Unique flow identifier
    -- ctx.status     (number) HTTP status code
    -- ctx.headers    (table)  Response headers
    -- ctx.body       (string) Response body or nil

    ctx.headers["X-Inspected-By"] = "hugin"

    return {
        headers = ctx.headers,
        status = ctx.status,
        body = ctx.body
    }
end
```

### OnFlowCapture

Called after a complete request-response pair has been recorded. Read-only -- cannot modify the flow.

Lua function: `on_flow_capture`.

```lua
function on_flow_capture(ctx)
    if hugin.string_contains(ctx.url, "/api/admin") then
        hugin.log("warn", "Admin API call detected: " .. ctx.url)
    end
end
```

### OnScanResult

Called when the active scanner produces a finding. Can enrich or filter findings.

Lua function: `on_scan_result`.

```lua
function on_scan_result(finding)
    -- finding.issue_type, finding.severity, finding.title, finding.description
    hugin.log("info", "Finding: " .. finding.title)
end
```

### PassiveCheck

Runs passive analysis on every captured flow. Return a table of findings.

Lua function: `passive_check`.

```lua
function passive_check(request, response)
    local findings = {}

    if response.headers["x-frame-options"] == nil then
        table.insert(findings, {
            issue_type = "missing-header",
            severity = "low",
            confidence = "certain",
            title = "Missing X-Frame-Options header",
            description = "The response does not include X-Frame-Options."
        })
    end

    return findings
end
```

### ActiveCheck

Runs active checks that can make outbound HTTP requests. Return a table of findings.

Lua function: `active_check`.

```lua
function active_check(request, response)
    local result = hugin.http_get(request.url .. "?test=1")
    if result.status == 500 then
        return {{
            issue_type = "error-based",
            severity = "medium",
            confidence = "tentative",
            title = "Server error on parameter injection",
            description = "Adding ?test=1 triggered a 500 response."
        }}
    end
    return {}
end
```

### Finding Fields

Findings returned by PassiveCheck and ActiveCheck must include:

- `issue_type` (string, required) -- Category identifier (e.g. "xss", "sqli", "custom")
- `severity` (string, required) -- One of: `critical`, `high`, `medium`, `low`, `info`
- `confidence` (string, required) -- One of: `certain`, `firm`, `tentative`
- `title` (string, required) -- Short summary
- `description` (string, required) -- Detailed explanation
- `evidence` (string, optional) -- Supporting evidence
- `remediation` (string, optional) -- Recommended fix

## API Reference

### Basic API (always available)

These functions are available to all extensions regardless of permissions:

**Logging:**

- `hugin.log(level, message)` -- Log a message. Levels: `error`, `warn`, `info`, `debug`, `trace`

**Encoding/Decoding:**

- `hugin.base64_encode(data)` -- Base64 encode a string
- `hugin.base64_decode(data)` -- Base64 decode a string
- `hugin.url_encode(data)` -- URL-encode a string
- `hugin.url_decode(data)` -- URL-decode a string
- `hugin.json_encode(table)` -- Serialize a Lua table to JSON string
- `hugin.json_decode(string)` -- Parse JSON string into a Lua table

**Hashing:**

- `hugin.hash_md5(data)` -- MD5 hash, returns hex string
- `hugin.hash_sha256(data)` -- SHA-256 hash, returns hex string

**String Utilities:**

- `hugin.regex_match(pattern, text)` -- Regex match with captures. Returns a table of captured groups or nil if no match.
- `hugin.regex_find_all(pattern, text)` -- Find all regex matches. Returns a table of tables (one per match).
- `hugin.string_contains(haystack, needle)` -- Returns true if haystack contains needle
- `hugin.string_starts_with(s, prefix)` -- Returns true if s starts with prefix
- `hugin.string_ends_with(s, suffix)` -- Returns true if s ends with suffix

### Guarded API (requires permissions)

These functions require specific permissions declared in `extension.json`. Calls without the required permission will raise an error and be audit-logged.

**FileSystem permission:**

- `hugin.read_file(path)` -- Read file contents as string
- `hugin.write_file(path, contents)` -- Write string to file
- `hugin.file_exists(path)` -- Check if a file exists (returns boolean)

**NetworkAccess permission:**

- `hugin.http_get(url, [headers])` -- HTTP GET request. Returns `{status, body, headers}`. 30-second timeout, follows up to 10 redirects.
- `hugin.http_post(url, body, [headers])` -- HTTP POST request. Returns `{status, body, headers}`.
- `hugin.http_request(method, url, [headers], [body])` -- Generic HTTP request for any method (GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS).
- `hugin.timed_http_request(method, url, [headers], [body])` -- HTTP request with precise timing. Does NOT follow redirects (for accurate timing measurement). 60-second timeout.

All HTTP functions accept self-signed certificates and return a table with `status` (number), `body` (string), and `headers` (table).

**SystemCommands permission:**

- `hugin.exec(command)` -- Execute a system command. This is dangerous and should be reviewed carefully before granting.

## Permissions

Permissions are declared in the manifest and enforced at runtime by the sandbox. Every guarded API call is permission-checked and audit-logged.

- **ReadFlows** -- Access captured HTTP flow data (read-only)
- **ModifyFlows** -- Modify HTTP requests and responses in transit (required for OnRequest/OnResponse hooks that return modifications)
- **NetworkAccess** -- Make outbound HTTP requests from within the extension
- **FileSystem** -- Read and write files on disk
- **SystemCommands** -- Execute system commands (dangerous -- review before granting)

## Sandbox

All extensions run in a sandboxed Lua 5.4 environment with the following safety limits:

- **Execution timeout:** 30 seconds per invocation (configurable)
- **Memory limit:** 64 MB per extension instance
- **Recursion depth:** 100 levels maximum
- **Instruction check interval:** Safety hook fires every 10,000 Lua instructions to check timeout and memory usage
- **Permission gating:** Guarded API calls fail with a clear error if the required permission is not declared
- **Audit logging:** Every filesystem access, network request, and system command is logged with the extension ID, operation, and result

The sandbox prevents:

- Infinite loops consuming CPU indefinitely
- Memory exhaustion from unbounded allocations
- Unauthorized access to system resources
- Extensions blocking the proxy pipeline

## Managing Extensions

### Via CLI

```bash
hugin plugin install https://github.com/user/hugin-plugin-name
hugin plugin list
hugin plugin remove my-extension
```

### Via MCP

```
extensions(action: "list")
extensions(action: "load", path: "/path/to/extension")
extensions(action: "enable", id: "my-extension")
extensions(action: "disable", id: "my-extension")
extensions(action: "reload", id: "my-extension")
extensions(action: "stats")
extensions(action: "test_hook", hook: "on_request", data: {...})
```

### Via REST API

```
GET  /api/extensions              List all extensions
GET  /api/extensions/{id}         Get extension details
POST /api/extensions/{id}/load    Load extension
POST /api/extensions/{id}/unload  Unload extension
POST /api/extensions/{id}/enable  Enable extension
POST /api/extensions/{id}/disable Disable extension
POST /api/extensions/{id}/reload  Reload extension code
GET  /api/extensions/stats        Extension statistics
POST /api/extensions/test-hook    Test a hook with sample data
```

## Example: JWT Token Logger

A complete extension that logs JWT tokens found in Authorization headers:

**extension.json:**

```json
{
  "id": "jwt-logger",
  "name": "JWT Token Logger",
  "version": "1.0.0",
  "author": "Example",
  "description": "Logs JWT tokens from Authorization headers",
  "entry_point": "main.lua",
  "hooks": ["OnRequest"],
  "permissions": ["ReadFlows"]
}
```

**main.lua:**

```lua
function on_request(ctx)
    local auth = ctx.headers["authorization"] or ctx.headers["Authorization"]
    if auth then
        local token = auth:match("Bearer%s+(.+)")
        if token then
            local parts = {}
            for part in token:gmatch("[^%.]+") do
                table.insert(parts, part)
            end
            if #parts >= 2 then
                -- Add padding for base64
                local padded = parts[2]
                local pad = 4 - (#padded % 4)
                if pad < 4 then
                    padded = padded .. string.rep("=", pad)
                end
                local payload = hugin.base64_decode(padded)
                if payload then
                    hugin.log("info", "JWT payload: " .. payload)
                end
            end
        end
    end
    return "continue"
end
```

## Example: CORS Header Checker

A passive check that flags overly permissive CORS configurations:

**extension.json:**

```json
{
  "id": "cors-checker",
  "name": "CORS Header Checker",
  "version": "1.0.0",
  "author": "Example",
  "description": "Flags permissive CORS configurations",
  "entry_point": "main.lua",
  "hooks": ["PassiveCheck"],
  "permissions": ["ReadFlows"]
}
```

**main.lua:**

```lua
function passive_check(request, response)
    local findings = {}
    local acao = response.headers["access-control-allow-origin"]
    local acac = response.headers["access-control-allow-credentials"]

    if acao == "*" and acac == "true" then
        table.insert(findings, {
            issue_type = "cors-misconfiguration",
            severity = "high",
            confidence = "certain",
            title = "Wildcard CORS with credentials",
            description = "Access-Control-Allow-Origin: * combined with "
                .. "Access-Control-Allow-Credentials: true. "
                .. "This is a browser-level protection bypass.",
            evidence = "ACAO: " .. tostring(acao) .. ", ACAC: " .. tostring(acac)
        })
    end

    return findings
end
```
