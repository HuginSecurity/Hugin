# Hugin

Hugin is a security intercepting proxy for web application penetration testing. It captures, inspects, and modifies HTTP/HTTPS traffic between your browser and target applications.

![Hugin HTTP History](./hugin-http-history.png)

## Key Features

- **Proxy** -- HTTP/1.1, HTTP/2, and WebSocket interception with automatic per-host TLS certificates
- **Repeater** -- Replay and modify requests with timing analysis and comparison
- **Intruder** -- Automated fuzzing with sniper, battering ram, pitchfork, and cluster bomb attack types
- **Scanner** -- 41 active + 35 passive vulnerability checks, built-in
- **Sequencer** -- Statistical analysis of session token randomness (entropy, FIPS, bit-level)
- **Decoder** -- Encode/decode chains across 20+ formats (Base64, URL, hex, HTML, JWT, and more)
- **Synaps** -- WASM-based vulnerability scanner with community module marketplace
- **Nerve** -- Passive response intelligence: parameter discovery, technology fingerprinting
- **RatRace** -- Race condition testing with single-packet, last-byte sync, and barrier modes
- **Oastify** -- Out-of-band detection with DNS, HTTP, SMTP, LDAP, FTP, and SMB listeners
- **Crawler** -- Recursive web crawler with headless browser support
- **YesWeHugin** -- Native YesWeHack integration: browse programs, import scope, submit reports
- **Lua Plugins** -- Extend Hugin with Lua 5.4 scripts for custom interception logic
- **MCP Server** -- Claude Desktop and Claude Code integration via 90+ MCP tools
- **Desktop GUI** -- Native cross-platform desktop app (macOS, Linux, Windows)
- **REST and GraphQL API** -- Full programmatic access to all features

## Quick Start

Download the latest release for your platform from [hugin.nu/download](https://hugin.nu/download), then run:

```bash
hugin
```

The proxy starts automatically on `127.0.0.1:8080`. Configure your browser to use that address as its HTTP/HTTPS proxy, trust the Hugin CA certificate, and you're intercepting traffic.

See the [Installation](./getting-started/installation.md) guide for all platforms.
