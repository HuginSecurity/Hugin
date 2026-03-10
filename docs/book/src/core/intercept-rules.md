# Intercept Rules

Intercept Rules is an automated traffic processing engine. Whereas the manual intercept queue requires you to be watching, Intercept Rules evaluate every flow silently in the background and apply actions automatically: forwarding, dropping, modifying headers, flagging for review, or queuing for manual inspection.

## Rule Structure

Each rule has:

- **name** / **description** -- labels for the UI and export
- **enabled** -- toggle without deleting
- **priority** -- higher values run first; rules are evaluated in priority order
- **group** -- optional string for organizing rules into named profiles
- **conditions** -- list of conditions to evaluate
- **condition_logic** -- `And` (all must match) or `Or` (any must match)
- **action** -- what to do when conditions are satisfied
- **apply_to** -- `Request`, `Response`, or `Both`

## Conditions

Each condition specifies:

- **field** -- what to inspect
- **operator** -- how to compare
- **value** -- the comparison value
- **case_sensitive** (default: false)
- **negate** -- if true, the condition is satisfied when the match fails (NOT logic)

### Match Fields

- **Host** -- the request hostname
- **Path** -- the URL path component
- **Url** -- the full URL
- **Method** -- HTTP method (GET, POST, etc.)
- **StatusCode** -- response status code (as string)
- **Header(name)** -- value of a named header
- **RequestBody** -- request body text
- **ResponseBody** -- response body text
- **ContentType** -- value of the Content-Type header
- **ContentLength** -- value of the Content-Length header (for numeric comparisons)
- **Cookie(name)** -- value of a named cookie
- **QueryParam(name)** -- value of a named URL query parameter
- **Any** -- matches anywhere in the entire request or response; useful with Exists to unconditionally trigger

### Match Operators

- **Contains** -- substring match
- **NotContains** -- substring absence
- **Equals** -- exact match
- **NotEquals** -- exact non-match
- **StartsWith** -- prefix match
- **EndsWith** -- suffix match
- **Regex** -- full regular expression
- **LessThan** / **GreaterThan** -- numeric comparison
- **Exists** -- the field exists regardless of value
- **NotExists** -- the field does not exist

## Actions

- **Forward** -- let traffic pass unchanged (useful as a default catch-all)
- **Drop** -- block the request or response; the client receives an error
- **Intercept** -- add the flow to the manual intercept queue for pause-and-inspect
- **Modify(modifications)** -- apply modifications and forward
- **Flag {color, note}** -- tag the flow with a color and optional note; does not block traffic
- **Log {message}** -- write a log entry; does not block traffic
- **Multi(actions)** -- execute multiple actions in sequence (e.g., flag and modify)

## Modifications

A Modify action takes a list of modification objects.

**Targets:**

- **Header(name)** -- a request or response header
- **QueryParam(name)** -- a URL query parameter
- **Cookie(name)** -- a cookie value
- **Body** -- the request or response body
- **Path** -- the URL path
- **Method** -- the HTTP method
- **StatusCode** -- the response status code
- **Host** -- the request hostname

**Operations:**

- **Set(value)** -- replace the entire field value
- **Remove** -- delete the field entirely
- **Replace {find, replace, regex}** -- find-and-replace within the current value
- **Append(value)** -- append to the current value
- **Prepend(value)** -- prepend to the current value

## Rule Groups

Rules are organized into named groups. A group has an `enabled` flag -- disabling a group disables all its rules at once. This makes it easy to switch between testing profiles (e.g., enable a "No Auth" group that strips authorization headers, disable it when done).

## Built-in Presets

Ready-to-use rules that cover common testing scenarios:

- **block_trackers()** -- drops requests to analytics and tracking domains (google-analytics, doubleclick, etc.)
- **flag_credentials()** -- flags requests with Authorization headers, login-path URLs, or body fields containing password/token/api_key
- **strip_security_headers()** -- removes CSP, X-Frame-Options, HSTS, X-Content-Type-Options, and X-XSS-Protection from responses
- **add_header(name, value)** -- injects a header into all requests
- **custom_user_agent(ua)** -- replaces the User-Agent on all requests
- **intercept_logins()** -- queues requests to login/auth paths for manual inspection
- **flag_api_keys()** -- flags responses containing AWS access keys, Stripe secret keys, or generic API key patterns
- **highlight_json()** -- flags JSON responses with a blue color
- **flag_large_responses(size_bytes)** -- flags responses larger than the given byte count
- **block_file_types(extensions)** -- drops requests for named file extensions (e.g., png, jpg, css)

## Examples

**Inject a testing header into every request:**
```json
{
  "name": "Add X-Test-Mode",
  "enabled": true,
  "priority": 10,
  "conditions": [{"field": "Any", "operator": "Exists", "value": ""}],
  "condition_logic": "And",
  "action": {"Modify": [{"target": {"Header": "X-Test-Mode"}, "operation": {"Set": "1"}}]},
  "apply_to": "Request"
}
```

**Drop static assets:**
```json
{
  "name": "Block Static",
  "enabled": true,
  "priority": 50,
  "conditions": [{"field": "Path", "operator": "Regex", "value": "\\.(css|js|png|jpg|gif|woff2)$"}],
  "condition_logic": "And",
  "action": "Drop",
  "apply_to": "Request"
}
```

**Flag responses over 1 MB:**
```json
{
  "name": "Large Response",
  "enabled": true,
  "priority": 5,
  "conditions": [{"field": "ContentLength", "operator": "GreaterThan", "value": "1048576"}],
  "condition_logic": "And",
  "action": {"Flag": {"color": "orange", "note": "Large response"}},
  "apply_to": "Response"
}
```

## MCP Tool

**`rules`** -- manage intercept rules.

Actions:

- `list` -- list all rules
- `get` -- get a rule by ID
- `create` -- create a new rule (requires name, conditions, rule_action)
- `update` -- modify an existing rule
- `delete` -- remove a rule
- `groups_list` -- list rule groups
- `groups_create` -- create a new group
- `groups_delete` -- delete a group
