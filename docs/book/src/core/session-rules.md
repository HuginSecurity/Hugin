# Session Rules

Session Rules automate token management across testing sessions. Instead of manually updating Authorization headers or cookies every time a session expires, you define rules that extract tokens from responses and inject them into subsequent requests automatically.

## The Problem

Penetration tests often last hours. Session tokens expire. Every time you need to re-authenticate, you interrupt your workflow. Session Rules solve this by detecting expiration, triggering a re-authentication macro, and propagating new tokens forward without manual intervention.

## Rule Structure

Each session rule contains:

- **name** -- a label for the rule
- **scope** -- a scope pattern limiting which URLs this rule applies to (host wildcard, URL prefix, or regex)
- **validation** -- optional configuration for detecting invalid sessions
- **reauth_macro** -- optional UUID of a macro to execute when the session expires
- **token_updates** -- list of extraction + injection pairs (where to find the token, where to put it)
- **enabled** / **priority** -- toggle and ordering; higher priority rules are evaluated first

## Detecting Session Expiry

Validation checks detect when a session is no longer valid:

- **StatusCode** -- response status is in a given list (e.g., `[401, 403]`); the most common check
- **ResponseContains** -- response body contains a string (e.g., `"session expired"`)
- **ResponseNotContains** -- response body no longer contains an expected string (e.g., `"authenticated"`)
- **HeaderContains** -- a specific header contains a pattern
- **Regex** -- a regex matches the response

One check type is active per rule.

## Token Sources

Where to extract a token from the response:

- **ResponseBody(regex)** -- regex with one capture group; the captured group is the token
- **ResponseHeader(header_name, regex)** -- token extracted from a response header using a regex
- **Cookie(cookie_name)** -- token is the value of a `Set-Cookie` header with the given name

## Token Targets

Where to inject the extracted token into subsequent requests:

- **RequestHeader(header_name)** -- sets the value of the named request header
- **Cookie(cookie_name)** -- sets the value of the named cookie in the Cookie header
- **QueryParam(param_name)** -- sets the value of a URL query parameter

## Built-in Presets

Four preset rules cover the most common patterns:

**JWT Bearer auth** -- detects 401/403 as invalid, extracts `access_token` from JSON response body, injects into `Authorization` header as `Bearer <token>`.

**Cookie-based session** -- detects 302/401/403, extracts the named cookie from `Set-Cookie`, injects into `Cookie` header.

**CSRF token** -- extracts the CSRF token from a hidden input field (`<input name="_csrf" value="...">`), injects into the named request header.

**OAuth2 refresh** -- detects 401, triggers the macro to get a new access token, extracts both `access_token` and `refresh_token` from the response body.

## How It Works

The session plugin hooks into the proxy pipeline:

- **On response** -- extracts tokens from every matching response and caches them in memory. If a response indicates session expiry, the plugin flags the macro for execution.
- **On request** -- if cached tokens exist, injects them into outgoing requests matching the rule scope.

Both auto-extract and auto-inject are enabled by default and can be disabled independently.

Cached tokens are held in a thread-safe map keyed by token name. Each new extraction overwrites the previous value, so token rotation is handled transparently.

## Configuration Example

```json
{
  "rules": [
    {
      "name": "API Auth",
      "scope": {"pattern": "*.api.example.com", "target": "host"},
      "validation": {
        "check_type": {"type": "StatusCode", "value": [401]}
      },
      "reauth_macro": "uuid-of-login-macro",
      "token_updates": [
        {
          "token_name": "access_token",
          "source": {"type": "ResponseBody", "regex": "\"access_token\":\"([^\"]+)\""},
          "target": {"type": "RequestHeader", "name": "Authorization"}
        }
      ],
      "enabled": true,
      "priority": 100
    }
  ],
  "auto_extract": true,
  "auto_inject": true
}
```

## MCP Tool

**`session`** -- session and authentication management.

Actions:

- `tokens` -- list currently cached tokens
- `status` -- session manager status
- `list_macros` -- list login macros
- `create_macro` -- create a new login macro
- `get_macro` -- get a specific macro
- `delete_macro` -- remove a macro
- `execute_macro` -- run a macro manually
- `refresh` -- force token refresh
