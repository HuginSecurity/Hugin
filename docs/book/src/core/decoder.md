# Decoder

Decoder is a multi-format encode/decode/hash/transform workbench with a built-in polyglot payload library. Paste a value, pick an operation, get the result. All operations are stateless -- nothing is stored or logged.

## Encode Operations

- **base64** -- standard Base64 with `=` padding
- **url** -- percent encoding
- **html** -- HTML entity encoding (`<` to `&lt;`, etc.)
- **unicode** -- Unicode escape sequences (`A` to `\u0041`)
- **hex** -- lowercase hex encoding of UTF-8 bytes
- **base32** -- RFC 4648 Base32 with `=` padding
- **ascii85** -- Adobe/PostScript Ascii85 encoding
- **rot13** -- Caesar cipher with 13-character rotation (symmetric: encode = decode)
- **binary** -- space-separated 8-bit binary representation per byte
- **octal** -- space-separated 3-digit octal representation per byte
- **gzip** -- gzip-compress the input, return Base64 of the compressed bytes
- **deflate** -- DEFLATE-compress the input, return Base64 of the compressed bytes
- **jwt_none** -- encode a JWT with `alg: none` and no signature

## Decode Operations

All encode operations have a corresponding decode operation. Additional notes:

- `gzip` and `deflate` expect Base64-encoded compressed data on input
- `binary` expects space-separated 8-bit groups
- `octal` expects space-separated 3-digit groups
- `unicode` replaces `\uXXXX` sequences with their code points
- `hex` strips spaces before parsing
- `jwt` decodes a JWT token (see JWT Decode below)

## Hash Algorithms

Hashing is one-way and returns a lowercase hex digest.

- **md5** -- MD5 (128-bit)
- **sha1** -- SHA-1 (160-bit)
- **sha256** -- SHA-256
- **sha512** -- SHA-512
- **sha3_256** -- SHA-3 256-bit
- **sha3_512** -- SHA-3 512-bit
- **blake2b** -- BLAKE2b
- **blake2s** -- BLAKE2s
- **ripemd160** -- RIPEMD-160
- **crc32** -- CRC-32 (non-cryptographic; useful for checksum verification)

## Transform Operations

Transforms manipulate text without encoding or decoding:

- **lowercase** / **uppercase** -- case conversion
- **reverse** -- reverse character order
- **length** -- return the byte count
- **trim** -- remove leading and trailing whitespace
- **remove_whitespace** -- remove all whitespace characters
- **remove_newlines** -- remove `\n` and `\r` characters

## JWT Decode

Splits a JWT at the `.` delimiters, Base64url-decodes the header and payload, and returns them as parsed JSON alongside the raw signature.

## JWT Forge

Constructs a new JWT from arbitrary header and payload JSON objects.

- **none** -- no signature; produces `header.payload.` with empty signature
- **hs256** -- HMAC-SHA256 with a provided secret key

The forged token is ready to paste into Repeater or an Intruder payload list.

## Chained Operations

The `chain` action applies a sequence of operations in order, passing the output of each step as the input to the next.

Operations are specified as strings with an optional prefix:

- `encode:base64` -- explicitly encode
- `decode:url` -- explicitly decode
- `base64` -- defaults to encode when no prefix is given

Each step is recorded in the response so you can see the intermediate value at every stage.

Example chain: `["decode:base64", "decode:url", "encode:html"]`

## Auto-Analysis

The `analyze` action inspects an unknown value and returns a confidence-ranked list of detected encodings:

- Base64 (0.9 confidence) -- length divisible by 4, valid character set
- URL encoding (0.95) -- presence of `%XX` sequences
- HTML entities (0.9) -- presence of `&amp;`, `&lt;`, or `&#`
- Hex (0.7) -- all hex-valid characters, even length
- Unicode escapes (0.95) -- presence of `\uXXXX` patterns
- JWT (0.95) -- three dot-separated Base64url segments
- Base32 (0.6) -- characters in `[A-Z2-7=]`

Also returns input length and Shannon entropy to help distinguish encoded data from random noise.

## Re-encode Conversion

The `reencode` action takes a value and attempts all pairwise conversions between base64, url, html, hex, and unicode. Returns every successful `from -> to` conversion path with intermediate decoded values.

## Polyglot Payloads

The `polyglot` action generates WAF bypass payloads from a library of 622 payloads across 66 vulnerability contexts:

XSS, SQLi, NoSQL injection, command injection, SSTI, XXE, SSRF, path traversal, LFI, RFI, LDAP injection, CRLF injection, open redirect, prototype pollution, deserialization, GraphQL injection, Log4j/Log4Shell, CSV injection, HTTP request smuggling, host header injection, CORS bypass, JWT attacks, file upload bypass, LLM/prompt injection, XPath injection, OAuth abuse, race conditions, email header injection, API abuse, and error disclosure.

Optional filters:

- **context** -- filter by vulnerability type (e.g., `xss`, `sqli`)
- **waf** -- apply WAF-specific evasion transformations (cloudflare, akamai, aws, modsecurity, imperva, f5, barracuda)
- **limit** -- maximum number of payloads to return (default 100)

## MCP Tool

**`decoder`** -- encode, decode, hash, transform, and generate payloads.

Actions: `encode`, `decode`, `chain`, `analyze`, `reencode`, `jwt_decode`, `jwt_forge`, `polyglot`, `operations`

**`smart_decode`** -- auto-detect and decode encoding chains layer by layer.

Actions: `detect`, `auto_decode`, `detect_and_decode`, `encodings`
