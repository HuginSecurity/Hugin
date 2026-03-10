# Nerve -- Passive Parameter Intelligence

Nerve is a passive analysis engine that reads HTTP traffic and produces actionable intelligence about what to test. It does not send any requests. It reads parameter names and values from flows you have already captured and maps them to vulnerability categories using a signal database of 714 patterns across 21 categories.

## What Nerve Does

Nerve answers two questions:

1. **Given requests with parameters, which parameters are worth attacking and for which vulnerability class?** This is the `analyze` mode -- it takes existing traffic and identifies high-value targets.

2. **Given a bare URL with no parameters, what parameters should I try adding?** This is the `discover` mode -- it classifies the endpoint type and suggests parameters to fuzz, prioritized by relevance.

## The 21 Vulnerability Categories

- `idor` -- IDOR/BOLA: `user_id`, `account_id`, `order_id`, `resource`, numeric IDs
- `sqli` -- SQL injection: `query`, `search`, `filter`, `sort`, `order_by`
- `ssrf` -- SSRF: `url`, `uri`, `endpoint`, `callback`, `webhook`, `fetch`
- `xss` -- XSS: `message`, `comment`, `content`, `html`, `text`, `markup`
- `cmdi` -- Command injection: `cmd`, `exec`, `command`, `shell`, `run`
- `lfi` -- Path traversal/LFI: `file`, `path`, `include`, `template`, `page`
- `ssti` -- Template injection: `template`, `render`, `view`, `layout`
- `auth` -- Authentication: `token`, `session`, `password`, `key`, `secret`, `apikey`
- `redir` -- Open redirect: `redirect`, `return_url`, `next`, `goto`, `continue`
- `mass` -- Mass assignment: `role`, `isAdmin`, `permission`, `privilege`, `admin`
- `xxe` -- XXE: XML bodies with `DOCTYPE`, `ENTITY`
- `debug` -- Debug endpoints: `debug`, `trace`, `verbose`, `test`, `dev`
- `graphql` -- GraphQL: `query`, `mutation`, `variables`, `operationName`
- `gateway` -- Gateway bypass: `X-Forwarded-For`, `X-Real-IP`, `X-Original-URL`
- `edge` -- Edge/CDN: `cf-ray`, `x-vercel-id`, `fastly-*`
- `cloud` -- Cloud metadata: `role`, `iam`, `bucket`, `region`
- `prototype` -- Prototype pollution: `__proto__`, `constructor`, `prototype`
- `ai_llm` -- AI/LLM: `prompt`, `model`, `temperature`, `system`, `instruction`
- `workflow` -- Workflow: `step`, `stage`, `transition`, `action`, `state`
- `saas` -- SaaS abuse: `stripe`, `payment`, `amount`, `price`

## Analyzing Captured Traffic

Pass URLs or Hugin flows to the analyzer. Each parameter in each request is matched against all signals for the appropriate context (query string, body, header, cookie, path).

For each match the output includes:

- The parameter name
- Its current value (if present)
- The matched category and signal name
- The context it appeared in (query, body, header, cookie, path)
- The confidence level (High, Medium, Low)
- Whether the value itself carries risk (e.g. a numeric ID, a URL value, `true` for a boolean admin flag)
- Any detected contextual pair
- The detected framework (WordPress, Spring, Keycloak, etc.)

## Confidence Adjustment

Signal confidence starts at the level defined in the signal database but is adjusted upward based on the actual parameter value:

- `user_id=42` -- Short numeric value for an IDOR signal upgrades confidence to High.
- `isAdmin=true` -- Boolean admin-true for a mass-assignment signal upgrades to High.
- `redirect=http://evil.com` -- URL value for an SSRF/redirect signal upgrades to High.
- `file=../../etc/passwd` -- Path traversal pattern in a value upgrades to High.
- `template={{7*7}}` -- Template syntax in a value for SSTI upgrades to High.

## Contextual Pairs

Nerve detects 20 high-risk parameter combinations that individually might be medium risk but together indicate a specific attack surface:

- `url` + `webhook` -- SSRF data exfiltration
- `token` + `redirect` -- OAuth token theft
- `model` + `prompt` -- LLM prompt injection
- `role` + `user_id` -- Privilege escalation
- `grant_type` + `client_id` -- OAuth flow manipulation
- `namespace` + `pod` -- Kubernetes access
- `bucket` + `key` -- S3 object manipulation
- `__proto__` + `exec` -- Prototype pollution RCE chain
- `cache_key` + `host` -- Cache poisoning
- `x_forwarded_host` + `cache` -- Web cache poisoning
- `password` + `email` -- Credential attack surface
- `query` + `variables` -- GraphQL injection
- `function` + `payload` -- Serverless RCE
- `success_url` + `stripe` -- Post-payment redirect hijack
- `trace_id` + `service` -- Distributed tracing abuse
- `service_id` + `upstream` -- Service mesh lateral movement
- `x_forwarded` + `internal` -- WAF/ACL bypass
- `x_original_url` + `admin` -- Path-based ACL bypass
- `constructor` + `prototype` -- Full prototype pollution chain
- `stripe` + `amount` -- Payment amount manipulation

When a pair is detected, each finding from those parameters carries the pair label.

## CamelCase Normalization

Parameters like `userId` are automatically normalized to `user_id` before matching. The original parameter name is preserved in the output. This prevents misses on APIs that use camelCase naming.

## Parameter Discovery for Bare URLs

When you have a list of endpoints with no existing parameters, Nerve can suggest what to add.

The endpoint classifier detects the type (api, auth, search, upload, admin, redirect, export, webhook, user, payment, integration, dev_staging, swagger) from URL structure. Technology indicators (Next.js, GraphQL, WordPress, Java, PHP, .NET, Node.js, Keycloak, cloud SaaS) are detected from path and hostname patterns.

Each endpoint type is mapped to high-priority and medium-priority vulnerability categories. Parameters are extracted from the signal database for those categories and returned in priority order, tagged with the context where they should be tested (query, body, header, cookie).

## Exclusion Rules

Nerve filters out noise automatically:

- External social/analytics domains (Twitter, Google Analytics, Facebook) are excluded entirely.
- Pagination parameters with small numeric values (`page=2`, `limit=10`) are excluded for LFI signals.
- Drupal CSS aggregation parameters (`css?include=...&delta=0`) are excluded for LFI.
- Static asset versioning (`app.js?v=1.2.3`) is excluded.
- Any of the above exclusions are overridden if the actual value carries a risk indicator (e.g. `?include=../../etc/passwd` still fires even on a Drupal CSS URL).

## Analyzing Proxy Flows Directly

Nerve accepts flows from the proxy store directly. This parses the full request including method, path segments, query string, body (JSON and form-urlencoded), headers, and cookies. JSON bodies are flattened -- `{"user": {"id": 42}}` produces a body parameter `user.id = 42` and also checks the leaf key `id` independently.

## Output Sorting

Results are sorted by confidence descending (High before Medium before Low), then by category, then by parameter name. High-confidence findings with risk-bearing values appear at the top.

## MCP Tool: `paramhunter`

The Nerve engine is exposed as the `paramhunter` MCP tool.

**Actions:**

- `analyze` -- Analyze URLs for parameter signals. Pass `urls` (array of URLs with query parameters).
- `analyze_flows` -- Analyze captured proxy flows. Pass `flow_ids` (array of UUIDs) or `host` (string) with optional `limit`.
- `discover` -- Suggest parameters for bare URLs. Pass `urls` (array of endpoints).
- `categories` -- List all 21 vulnerability categories with signal counts.
- `info` -- Get detailed info for a specific category including all signals. Pass `category` (short name).
- `stats` -- Total signal counts across all categories.

## Integration with Flow Details

Nerve findings are automatically included when you retrieve flow details via the `flow_detail` MCP tool. The `nerve_findings` field on each flow shows which parameters matched which vulnerability categories, without requiring a separate tool call.

Aggregated Nerve statistics for the current project are available via the `intelligence` MCP tool with the `nerve_findings` and `nerve_stats` actions.
