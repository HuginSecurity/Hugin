# Match and Replace

Match and Replace rules transform HTTP traffic as it flows through the proxy. Each rule is evaluated against every request or response in real time before the flow is stored or forwarded. Use this for quick, pattern-based modifications -- stripping headers, injecting values, or rewriting URLs on the fly.

## Rule Structure

Every rule has:

- **name** -- a label shown in the UI and logs
- **enabled** -- toggle the rule on or off without deleting it
- **target** -- which part of the traffic to match against
- **pattern** -- the string or regular expression to find
- **replacement** -- what to put in place of the match; empty string removes the matched content
- **is_regex** -- if true, pattern is compiled as a regular expression and replacement may use capture group references (`$1`, `$2`, etc.)

Rules are applied in order. If a rule matches and modifies traffic, subsequent rules still run and may match on already-modified content.

## Match Targets

**Request targets** (applied before the request is forwarded to the server):

- **request_header {name}** -- matches the value of a specific request header (case-insensitive lookup). Only the value is modified, not the header name.
- **request_body** -- matches the raw request body as UTF-8 text. Binary bodies are skipped.
- **request_url** -- matches the full request URL string.
- **request_first_line** -- matches the entire HTTP request line (method, path, version).

**Response targets** (applied before the response is forwarded to the browser):

- **response_header {name}** -- matches the value of a specific response header.
- **response_body** -- matches the response body as UTF-8 text.
- **response_first_line** -- matches the HTTP response status line.

**Additional operations available in Quick Rules:**

- **Add request header** -- inject a new header into outgoing requests
- **Remove request header** -- strip a header from outgoing requests
- **Add response header** -- inject a header into incoming responses
- **Remove response header** -- strip a header from incoming responses

## Literal vs Regex

With `is_regex: false`, the pattern is a literal string and every occurrence is replaced.

With `is_regex: true`, the pattern is compiled as a regular expression. All matches are replaced (equivalent to `replace_all`). If the regex is invalid, the rule is silently skipped.

## Quick Rules vs Advanced Rules

The Match and Replace view has two tabs:

**Quick Rules** -- a flat, inline-editable table in the style of Burp's Match and Replace panel. Each row is a single match/replace pair with a target type, pattern, replacement, and regex toggle. Best for simple modifications.

**Advanced Rules** -- full Intercept Rules with complex condition logic, multiple conditions per rule, and multi-action support. See [Intercept Rules](./intercept-rules.md) for the full reference.

## Practical Examples

**Strip the Authorization header from all requests** (test unauthenticated behavior):
```json
{
  "target": "request_header",
  "name": "Authorization",
  "pattern": ".*",
  "replacement": "",
  "is_regex": true
}
```

**Replace a CORS origin:**
```json
{
  "target": "request_header",
  "name": "Origin",
  "pattern": "https://attacker.com",
  "replacement": "https://trusted.example.com",
  "is_regex": false
}
```

**Remove Content Security Policy from responses:**
```json
{
  "target": "response_header",
  "name": "Content-Security-Policy",
  "pattern": ".*",
  "replacement": "",
  "is_regex": true
}
```

**Rewrite API version in URLs:**
```json
{
  "target": "request_url",
  "pattern": "/api/v1/",
  "replacement": "/api/v2/",
  "is_regex": false
}
```

## Integration with the Plugin Bus

Match and Replace is implemented as a plugin in the proxy pipeline. On each request, the plugin iterates all enabled request-targeting rules. On each response, it iterates response-targeting rules. Rules are loaded from the store on startup and can be updated at runtime without restarting the proxy.

## API

```
GET    /api/match-replace/rules
POST   /api/match-replace/rules          Create a rule
PUT    /api/match-replace/rules/{id}     Update a rule
DELETE /api/match-replace/rules/{id}     Delete a rule
POST   /api/match-replace/rules/{id}/toggle  Enable/disable
```
