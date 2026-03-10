# Comparer

Comparer measures the difference between two HTTP responses. It is the foundation for differential testing -- detecting blind SQL injection, blind XSS, and other vulnerabilities where the application produces subtly different output depending on the injected value.

## How It Works

Comparer uses the Gestalt Pattern Matching algorithm to compute a similarity ratio between 0.0 (completely different) and 1.0 (identical). It operates on response body text after applying normalization options.

Beyond the similarity ratio, Comparer identifies specific differences and categorizes them:

- **StatusCode** -- the HTTP status codes differ
- **Header(name)** -- a specific header value differs
- **Body(offset)** -- a difference at an approximate character position in the body
- **BodyLength** -- the bodies differ only in length
- **ResponseTime** -- response times differ beyond a configurable threshold

Each difference has a type: Added, Removed, Changed, or LengthChange.

## Comparison Options

- **normalize_whitespace** (default: true) -- collapse whitespace before comparison; reduces noise from reformatted responses
- **ignore_case** (default: false) -- case-insensitive comparison
- **excluded_headers** -- headers excluded from comparison; by default: `date`, `set-cookie`, `x-request-id`, `x-correlation-id`, `x-trace-id`, `etag`, `last-modified`, `age`, `expires`
- **compare_timing** (default: false) -- include response time in the diff
- **timing_threshold_ms** (default: 1000) -- timing differences below this value are not flagged

## Input Formats

Both left and right sides accept either:

- A **flow ID** -- Comparer fetches the stored response from the database
- **Raw HTTP response text** -- paste the full response including status line, headers, and body

You can mix the two: compare a stored flow against a freshly-pasted response.

## Blind Injection Detection

The `blind_detect` action takes three responses: a baseline, a "true condition" response, and a "false condition" response. It computes:

- **true_similarity** -- how similar the true response is to the baseline
- **false_similarity** -- how similar the false response is to the baseline
- **similarity_gap** -- the difference between the two scores

If the gap is large enough (true response very similar to baseline, false response noticeably different), `is_vulnerable` is set to true along with a confidence score.

### Typical blind SQLi workflow

1. Capture the baseline response to a normal request.
2. Send a true-condition payload (e.g., `AND 1=1`) and capture that response.
3. Send a false-condition payload (e.g., `AND 1=2`) and capture that response.
4. Submit all three flow IDs to `blind_detect`.
5. If `is_vulnerable: true`, Comparer has measured a statistically significant difference.

## Similarity Score

The similarity ratio answers: what fraction of the content is shared between the two responses? A score of 1.0 means identical after normalization. A score of 0.0 means nothing is shared.

For blind injection testing, the relevant threshold is not the absolute values but the gap between the true and false conditions. The baseline anchors the comparison.

## Lightweight Similarity

The `similarity` action accepts two flow IDs and returns only the similarity ratio without the full difference list. Use this when comparing large numbers of responses quickly and only the score matters.

## MCP Tool

**`comparer`** -- response comparison for blind vulnerability detection.

Actions:

- `compare` -- compare two responses by flow_id or raw content
- `blind_detect` -- detect blind SQLi/XSS by comparing baseline/true/false responses
- `similarity` -- calculate similarity score between two flows

## API

```
POST /api/comparer/compare
{
  "left_flow_id": "uuid",
  "right_flow_id": "uuid",
  "options": {"normalize_whitespace": true, "ignore_case": false}
}

POST /api/comparer/blind-detect
{
  "baseline_flow_id": "uuid",
  "true_flow_id": "uuid",
  "false_flow_id": "uuid"
}

GET /api/comparer/similarity?left=uuid&right=uuid
```
