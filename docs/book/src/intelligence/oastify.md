# Oastify -- Out-of-Band Detection

Oastify is Hugin's built-in OOB (out-of-band) callback server. It listens for DNS, HTTP, HTTPS, SMTP, LDAP, FTP, and SMB interactions triggered by payloads injected during a scan. When a payload causes a server to make an outbound connection, Oastify receives and records that interaction, confirming blind vulnerabilities that produce no visible difference in the HTTP response.

## Why OOB Matters

Many vulnerability classes -- blind SSRF, blind XXE, blind SQL injection with DNS exfiltration, LDAP injection, Java deserialization -- cannot be detected by analyzing the HTTP response alone. The vulnerable server makes an outbound connection. Without a listener to receive that connection, the finding is unconfirmable. Oastify closes this gap by providing an instrumented listener on a domain you control.

## Protocol Listeners

Oastify starts seven listeners when launched:

- **DNS** (port 53) -- Receives A/AAAA/TXT queries. The most universally reachable protocol -- DNS egress is rarely blocked.
- **HTTP** (port 80) -- Receives HTTP GET/POST requests. Useful for SSRF and webhook redirect payloads.
- **HTTPS** (port 443) -- Same as HTTP with TLS. Required when the target application enforces HTTPS-only URLs.
- **SMTP** (port 25) -- Receives email delivery attempts. Useful for email header injection and server-side mail triggers.
- **LDAP** (port 389) -- Receives LDAP bind and search requests. The primary OOB vector for Log4Shell-class JNDI payloads.
- **FTP** (port 21) -- Receives FTP connection attempts. Some legacy XXE and SSRF payloads use FTP URIs.
- **SMB** (port 445) -- Receives UNC path resolution attempts. Windows-side SSRF and XXE payloads that reference `\\server\share` paths.

Each listener records the source IP, source port, protocol, timestamp, and protocol-specific detail (DNS query name, HTTP path, LDAP DN, etc.).

## Payload Generation

The `PayloadGenerator` produces correlation-tagged payloads for any of the seven protocols. Each payload contains a short correlation ID embedded in a subdomain or path:

```text
DNS:    <corr_id>.<your-domain>
HTTP:   http://<corr_id>.<your-domain>/
HTTPS:  https://<corr_id>.<your-domain>/
SMTP:   attacker@<corr_id>.<your-domain>
LDAP:   ldap://<corr_id>.<your-domain>/dc=<corr_id>
FTP:    ftp://<corr_id>.<your-domain>/
SMB:    \\<corr_id>.<your-domain>\share
```

The correlation ID is configurable: alphanumeric (default), hex, or base64. Length defaults to 8 characters.

`generate_all` returns payloads for all seven protocols in a single call. The scanner uses this when injecting into parameters that support multiple URI schemes.

## Correlation Flow

Every payload embeds a unique correlation ID. When Oastify receives an interaction, it extracts that ID from the DNS query name, HTTP Host header, or LDAP DN and matches it against the active payload set. The scanner marks the corresponding finding as confirmed when a match arrives.

The correlation lookup is available to the scanner (built-in checks) and to Synaps WASM modules via the `Context` trait:

```rust,ignore
let payload = ctx.oastify_dns(Some("my-check"))?;
// Inject payload.payload into the target request
// ...
if ctx.oastify_was_triggered(&payload.correlation_id) {
    return Ok(CheckResult::vulnerable(Confidence::Confirmed)
        .with_message("OOB DNS interaction received"));
}
```

## Configuration

Oastify requires a domain you control with a wildcard DNS record pointing to your Oastify server's IP (`*.your-domain.com -> server-ip`).

## Standalone Server Mode

`hugin-oastify-server` is a separate binary that runs just the Oastify listeners without the full proxy. Deploy it on a VPS or cloud instance reachable from your targets:

```bash
hugin-oastify-server --domain oob.example.com --bind 0.0.0.0
```

The standalone server exposes the same HTTP API as the embedded Oastify. Point Hugin's `callback_domain` setting at your VPS domain and it will poll the server for interactions via its API. This lets the proxy run locally while the callback endpoint is publicly reachable.

## Integration with the Active Scanner

When `callback_domain` is set in `ScanConfig`, the built-in checks that support OOB (SSRF, XXE, LDAP injection, JWT attacks, deserialization, stored XSS) automatically include Oastify payloads. The scanner marks those findings as triggered immediately in the finding list. To confirm them, poll Oastify for the correlation ID embedded in the payload.

Checks without a `callback_domain` fall back to error-based detection only. OOB confirmation upgrades finding confidence to `Confirmed`.

## Interactions

Each received interaction is stored with:

- `id` -- Unique interaction UUID
- `correlation_id` -- Extracted from the payload
- `protocol` -- DNS, HTTP, HTTPS, SMTP, LDAP, FTP, or SMB
- `source_ip` and `source_port` -- Where the callback originated
- `timestamp` -- When Oastify received it
- `details` -- Protocol-specific data (DNS query type and name, HTTP method and path, LDAP operation and DN, etc.)

All interactions are queryable by correlation ID or by time range.

## MCP Tool: `oastify`

**Actions:**

- `start` -- Start the Oastify server with `domain` and `external_ip`.
- `stop` -- Stop the server.
- `status` -- Get server status (running/stopped, listeners active).
- `domain` -- Get the configured callback domain.
- `stats` -- Interaction statistics (count by protocol, recent activity).
- `generate_payload` -- Generate a single payload for a specific protocol (`dns`, `http`, `https`, `smtp`, `ldap`, `ftp`, `smb`).
- `batch_payloads` -- Generate payloads for multiple protocols at once.
- `all_payloads` -- Generate payloads for all seven protocols.
- `list_payloads` -- List all generated payloads.
- `get_payload` -- Get a specific payload by ID.
- `interactions` -- List received interactions, optionally filtered by correlation ID or protocol.
- `interaction` -- Get a specific interaction by ID.
- `delete_interaction` -- Delete a specific interaction.
- `acknowledge` -- Mark an interaction as acknowledged (confirmed by the operator).
- `acknowledge_bulk` -- Bulk-acknowledge interactions.
