<p align="center">
  <img src="assets/hugin-raven.png" alt="Hugin" width="120">
</p>

<h1 align="center">Hugin</h1>

<p align="center">
  Security intercepting proxy for web application penetration testing.<br>
  Built with Rust. No JavaScript. One binary.
</p>

<p align="center">
  <a href="https://hugin.nu">Website</a> &middot;
  <a href="https://hugin.nu/download">Download</a> &middot;
  <a href="https://hugin.nu/docs">Documentation</a> &middot;
  <a href="https://hugin.nu/pricing">Pricing</a>
</p>

---

## 🎉 v0.1.0 is live

The first public release is out. [Download for your platform below](#download) or grab it from the [Releases page](../../releases/tag/v0.1.0).

All binaries are **Ed25519 signed**. Verify before running.

---

## What is Hugin?

Hugin is a security proxy built for bug bounty hunters and penetration testers. It gives you everything you need to start hunting on day 1 — free, with no account required.

**Community (free, forever):**
- MITM Proxy (HTTP/1.1, HTTP/2, WebSocket) with HTTP/3 detection (Alt-Svc tagging)
- HTTP/3 outbound client — Repeater and MCP can send QUIC requests, probe Alt-Svc, fingerprint QUIC implementations (full HTTP/3 MITM landing in v0.2.0)
- Active Scanner (42 checks: SSRF, SQLi, SSTI, XSS, JWT, LDAP, BOLA, path traversal, HTTP smuggling, race conditions, prototype pollution, cache poisoning, and more)
- Passive Scanner (36 checks: security headers, TLS, CSP, CORS, info disclosure, DOM XSS sources, stack traces, and more)
- Intruder (20 payload types, 15 processing rules, grep match/extract)
- Repeater
- Sequencer, Decoder, Comparer, Site Map
- Scriptable UI automation (navigate views, select flows, switch panes, highlight, screenshot — all from MCP)
- 130+ MCP tools (connect Claude, Cursor, or any MCP-compatible AI agent)

**Pro (5 EUR/month — for everyone):**
- Race condition engine (60+ modules, single-packet, last-byte sync)
- Synaps WASM modules (community scanner modules, sandboxed)
- Lua extensions (modify live traffic with scripts)
- E2E encrypted real-time collaboration
- Multi-project workspaces
- 56 offensive vurl MCP tools (HTTP smuggling, deserialization, SSRF chains, WAF evasion, AI agent exploitation, and more)

No subscriptions. No auto-renewal. Pay when you need it.
Researcher, pentester, hacker — we don't differentiate. Everyone's welcome.

## This Repository

This is the **community hub** for Hugin. The source code is not hosted here.

Use this repo to:

- **Report bugs** — [Open a bug report](../../issues/new?template=bug_report.yml)
- **Request features** — [Open a feature request](../../issues/new?template=feature_request.yml)
- **Discuss** — [Join discussions](../../discussions) for questions, ideas, and community chat
- **Track releases** — [Releases](../../releases) for changelogs and download links

## Students

If you have a **GitHub Student Developer Pack**, you get **12 months of Pro for free**. No forms, no proof uploads — GitHub already verified you.

Claim yours at [hugin.nu/students](https://hugin.nu/students).

## Download

Latest release: **[v0.1.0](../../releases/tag/v0.1.0)** (2026-04-18)

**macOS**
- [Apple Silicon (.dmg)](../../releases/download/v0.1.0/hugin-desktop-darwin-aarch64.dmg)
- [Intel (.dmg)](../../releases/download/v0.1.0/hugin-desktop-darwin-x86_64.dmg)
- CLI only: [aarch64](../../releases/download/v0.1.0/hugin-cli-darwin-aarch64.tar.gz) · [x86_64](../../releases/download/v0.1.0/hugin-cli-darwin-x86_64.tar.gz)

**Linux (x86_64)**
- [AppImage](../../releases/download/v0.1.0/hugin-desktop-linux-x86_64.AppImage) — chmod +x and run, any distro
- [.deb](../../releases/download/v0.1.0/hugin-desktop-linux-x86_64.deb) — Debian/Ubuntu
- [Tarball](../../releases/download/v0.1.0/hugin-desktop-linux-x86_64.tar.gz) — portable
- CLI only: [tarball](../../releases/download/v0.1.0/hugin-cli-linux-x86_64.tar.gz)

**Coming in v0.1.1**
- Linux aarch64
- Windows x86_64

All binaries are **Ed25519 signed**. Verify with `hugin verify <file>` or at [hugin.nu/verify](https://hugin.nu/verify).
Public key: `a61ff9262c4509a7879ddaa5a8d86345ef805f6ddced28b097bf58dae270618b`

## Privacy

- No telemetry. No analytics. No crash reporting.
- Your traffic never leaves your machine.
- Accounts are anonymous IDs — no email, no password, no recovery.
- Payments via Stripe or Bitcoin/Monero (BTCPay). No KYC.

## Links

- [hugin.nu](https://hugin.nu) — Official website
- [hugin.nu/docs](https://hugin.nu/docs) — Documentation
- [hugin.nu/about](https://hugin.nu/about) — Why Hugin exists

## License

Hugin is proprietary software. The Community tier is free to use. See [hugin.nu/pricing](https://hugin.nu/pricing) for details.
