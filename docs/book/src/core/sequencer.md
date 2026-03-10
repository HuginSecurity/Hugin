# Sequencer

Sequencer analyzes the statistical quality of tokens -- session IDs, CSRF tokens, password-reset links, API keys, or any value an application generates. It tells you whether the tokens are cryptographically random or follow a pattern that an attacker could predict.

## Quick Start

1. Identify a token you want to analyze (session cookie, CSRF token, etc.).
2. Start a capture session targeting that token, or paste a list of collected tokens.
3. Collect at least 100 tokens (2,500+ bytes for FIPS tests).
4. Run the analysis.
5. Review the quality grade and individual test results.

## Analysis Pipeline

### Shannon Entropy

Entropy is calculated in bits per byte over the byte distribution of all tokens concatenated. A perfect random byte sequence scores 8.0 bits. Printable ASCII strings are bounded by the character set -- a hex-only token tops out around 4.0 bits (16 symbols). Entropy below 4.0 bits is a strong signal of poor randomness.

### Bit-Level Analysis

Each of the 8 bit positions (0-7) is analyzed independently. For each position, Sequencer calculates the probability it is set to 1 across all token bytes. An ideal random generator produces 0.5 for every position. Any position more than 5% away from 0.5 is flagged as biased. Common findings: ASCII bit 7 is always 0 for printable strings; base64 tokens constrain multiple bit positions.

### FIPS 140-2 Tests

All four FIPS 140-2 statistical randomness tests are implemented. These require at least 2,500 bytes (20,000 bits) of token data -- collect enough samples before interpreting FIPS results.

**Monobit Test** -- counts the number of 1 bits in a 20,000-bit sample. Must fall between 9,725 and 10,275.

**Poker Test** -- divides bits into 5,000 four-bit segments and applies a chi-square test over the 16 possible patterns. Statistic X must be between 2.16 and 46.17.

**Runs Test** -- counts consecutive runs of 0s and 1s in six length categories (1 through 6+). Each category must fall within bounds derived from the FIPS standard.

**Long Runs Test** -- no run of consecutive identical bits may exceed 25 bits. Tokens with long static prefixes commonly fail here.

All four tests must pass for `overall_pass` to be true.

### Pattern Detection

Sequencer scans the token list for structural patterns:

- **Common prefix** -- all tokens share a leading substring of 2+ characters. Prefixes longer than 4 characters are rated High severity.
- **Common suffix** -- same analysis for trailing substrings.
- **Sequential** -- tokens parse as integers forming an arithmetic sequence. Rated Critical.
- **Repeating** -- tokens that appear more than once. More than half being duplicates is Critical.
- **Charset restriction** -- all tokens use only one character class (lowercase, uppercase, digits, hex). Hex-only is informational; other single-class restriction is Medium.

### Correlation Analysis

When 100+ bytes of data are available, Sequencer calculates the serial correlation coefficient and auto-correlation at up to 20 lags. A coefficient above 0.1 indicates statistically significant correlation. Correlation above 0.3 reduces the quality score significantly.

## Quality Grades

The analysis produces a single quality grade backed by a numeric score (0-100):

- **Excellent** (>= 85) -- suitable for cryptographic use
- **Good** (>= 65) -- passes all standard tests
- **Fair** (>= 45) -- minor concerns
- **Poor** (>= 25) -- not suitable for security purposes
- **Critical** (< 25) -- highly predictable

Scoring deductions: low entropy (-10 to -40), failed FIPS tests (-30 for overall fail, up to -15 for individual), significant correlation (-20), detected patterns (-8 to -25 depending on severity), biased bit positions (-10 to -20).

## Capture Modes

Tokens can reach Sequencer in two ways:

- **Live capture** -- start a capture session that sends repeated requests and extracts the target token from each response automatically.
- **Manual paste** -- paste a list of tokens directly or submit them via the API.

## Comparing Token Sets

The `compare` action lets you compare two sets of tokens side by side -- useful for checking whether a fix improved randomness quality or whether tokens from different endpoints share weaknesses.

## Output

The analysis report includes:

- Quality grade and numeric score
- Entropy value in bits
- Bit-position analysis showing bias per position
- FIPS test results with pass/fail, computed value, and expected threshold range
- Correlation coefficient and auto-correlation values
- Detected pattern list with severity, type, and confidence
- Entropy distribution chart (ASCII visualization per 64-byte block)

## MCP Tool

**`sequencer`** -- token randomness analysis.

Actions:

- `capture` -- start collecting tokens from repeated requests
- `tokens` -- poll collected tokens from an active session
- `stop` -- stop capture
- `analyze` -- run statistical analysis on collected tokens
- `list` -- list active sessions
- `delete` -- remove a session
- `status` -- get session status with all tokens
- `compare` -- compare two token sets
- `export` -- export session with full analysis report
