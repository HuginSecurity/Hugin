# Intruder

Intruder is Hugin's automated attack engine. Mark positions in a request with injection markers (`§name§`), choose payload generators and processors, and Intruder fires every payload through those positions while recording response status codes, lengths, timing, and pattern matches.

## Quick Start

1. Capture a request in the proxy.
2. Send it to Intruder.
3. Mark injection points by wrapping values with `§` markers: `/api/users/§id§`.
4. Configure payload generators for each position.
5. Start the attack.
6. Review results, filtering by status code, response length, or timing anomalies.

## Payload Generators (19 types)

Each generator produces a lazy iterator of strings. Generators do not materialize the full list in memory unless required.

**SimpleList** -- a fixed list of strings you provide inline. The most common generator for custom wordlists.

**Numbers** -- a numeric range with step. Supports zero-padded formatting (`{:04}`) and negative steps for descending ranges. Example: `start=1, end=100, step=1` produces `1` through `100`.

**BruteForcer** -- all permutations of a character set between a minimum and maximum length. Built-in charsets: lowercase, uppercase, digits, alphanumeric, hex. Maximum string length is capped at 12; maximum total payloads is 1,000,000.

**Dates** -- a date range with a strftime format string. Useful for brute-forcing date-based tokens or IDs. Example: `start=2024-01-01, end=2024-12-31, format=%Y%m%d`.

**CharacterSubstitution** -- generates all combinations of a base string where specific characters are replaced by alternatives. Includes a built-in leet-speak rule set (`a->4/@`, `e->3`, etc.) or define your own substitution map.

**CharFrobber** -- applies character manipulations to a base string: toggle case per-character, toggle entire string, increment/decrement character codes, reverse, shift by offset. Produces the original plus all derived variants.

**BitFlipper** -- flips bits in a base value systematically. Five modes: SingleBit (one bit at a time), SingleByte (XOR each byte with 0xFF), BitPairs (flip consecutive bit pairs), BytePairs (flip consecutive byte pairs), Nibble (flip each 4-bit nibble).

**NullPayloads** -- emits N copies of an empty string, or a fixed set of 14 null-like values (`null`, `NULL`, `nil`, `None`, `undefined`, `NaN`, `0`, `-1`, `false`, `\0`, `\x00`, `%00`, `{{null}}`, and empty string).

**UsernameGenerator** -- generates username variations from first/last name pairs. Patterns: `first.last`, `first_last`, `flast`, `firstl`, `first`, `last`, `first.last1..99`. Includes a CommonNames pattern that emits 25 common admin/system usernames.

**RuntimeFile** -- reads payloads line by line from a file at attack time. Blank lines and comment lines (starting with `#`) are skipped. Validates the file path against an optional base directory to prevent path traversal.

**ExtensionGenerated** -- a custom generator backed by a Lua extension closure. Lets extensions inject dynamic payload sequences at runtime.

**CharacterBlocks** -- generates strings of a single character repeated at varying lengths. Useful for buffer overflow testing and length-based fuzzing. Configurable character, min/max length, and step.

**CustomIterator** -- generates combinations from multiple word lists joined by a separator. Useful for producing structured strings like `admin-dev-01`, `admin-staging-02`.

**IllegalUnicode** -- generates illegal and malformed Unicode sequences for testing parser robustness. Categories include overlong UTF-8, surrogate pairs, BOM variations, and null bytes. Output is percent-encoded for safe transport.

**CaseVariant** -- generates case variations of base words: uppercase, lowercase, proper case, inverted case. Deduplicates results since different modes may produce identical output for certain inputs.

**OastifyPayloads** -- generates out-of-band (OOB) callback payloads using the built-in Oastify system. Each payload contains a unique correlation ID for tracking interactions. Supports DNS, HTTP, HTTPS, SMTP, LDAP, FTP, and SMB protocols.

**EcbBlockShuffler** -- rearranges ciphertext blocks to test ECB-mode encryption vulnerabilities. Operations include removing blocks, duplicating blocks, swapping pairs, reversing order, and generating all permutations. Supports hex and base64 encoded ciphertext with 8-byte (DES) or 16-byte (AES) block sizes.

**CopyOtherPayload** -- copies the payload from another position. Resolved during generation; used when two positions must receive the same value.

## PayloadSet Combination Modes

When an attack uses multiple payload positions, generators are combined:

- **concat** -- all payloads from generator 1, then all from generator 2 (serial)
- **zip** -- pairs payloads position by position; stops when the shortest generator is exhausted
- **product** -- Cartesian product of all generators (every combination). Capped at 10,000,000 total combinations.

## Payload Processors (15 types)

Processors transform each payload before injection. Chain multiple processors to build complex transformations.

- **AddPrefix** / **AddSuffix** -- prepend or append a fixed string
- **MatchReplace** -- regex or literal find/replace within the payload
- **UrlEncode** / **UrlDecode** -- percent encoding; configurable to encode all characters or only special ones
- **HtmlEncode** / **HtmlDecode** -- HTML entity encoding
- **Base64Encode** / **Base64Decode** -- standard Base64
- **Hash** -- hash the payload with MD5, SHA-1, SHA-256, or SHA-512
- **CaseModification** -- uppercase, lowercase, or toggle case
- **Substring** -- extract a substring by start and optional end index
- **ReverseString** -- reverse the payload
- **SkipIfMatch** -- drop the payload entirely if it matches a pattern (filter)
- **InvokeExtension** -- pass the payload to a Lua extension function for arbitrary transformation

## Response Analysis: Grep Match

Attach grep matchers to flag responses containing (or missing) specific content:

- **Contains** -- substring match, optionally case-sensitive
- **NotContains** -- inverse match
- **Regex** -- regular expression match with a 1 MB size limit to prevent ReDoS

Match results include the patterns that fired and byte offsets within the response body.

## Response Analysis: Grep Extract

Extractors pull values out of responses:

- **from_regex(name, pattern, group)** -- captures a named group from each response
- **from_delimiters(name, start, end)** -- extracts content between two delimiter strings

Extracted values are stored per-result and visible in the results table.

## Result Filtering and Sorting

Results can be filtered by:

- Exact status code or status code range (e.g., `2xx`)
- Response length (min, max)
- Response time (min, max)
- Whether a grep match fired
- Whether a specific key was extracted
- Errors only or exclude errors

Results are sortable by payload, request ID, status code, response length, or response time.

## Grep Local

The `grep_local` action provides fast Rust-native regex searching across attack results. Supports filters for status codes, response time ranges, length ranges, field selection (body, headers, status), and result inversion.

## Resource Pool

Each attack runs with a configurable concurrency limit controlling how many requests are in-flight simultaneously. Rate limiting is enforced between requests. Attacks can be paused and resumed.

## CSV Export

Any result set can be exported to CSV. The export includes request ID, payload, status code, response length, response time, grep match flag, matched patterns, extracted values, and error message.

## MCP Tool

**`intruder`** -- attack automation engine.

Actions: `start`, `run` (standalone, no proxy required), `list`, `get`, `status`, `pause`, `resume`, `cancel`, `delete`, `results`, `grep_local`, `grep_data`, `processing`, `export`

The `run` action executes an attack inline and returns results directly -- useful for quick one-off fuzzing from MCP without managing attack state.

## API

```
POST /api/intruder/attacks              Start a new attack
GET  /api/intruder/attacks              List all attacks
GET  /api/intruder/attacks/{id}         Get attack details
GET  /api/intruder/attacks/{id}/results Get results
POST /api/intruder/attacks/{id}/pause   Pause
POST /api/intruder/attacks/{id}/resume  Resume
POST /api/intruder/attacks/{id}/cancel  Cancel
```
