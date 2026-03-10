# Telemetry

Hugin includes an optional, anonymous telemetry system to help prioritize development.

## Privacy

- **Off by default** -- must be explicitly enabled
- **No PII** -- no URLs, IPs, hostnames, request content, findings, or credentials
- **No tracking** -- installation ID is a random UUID, not linked to any account
- **Open source** -- the telemetry code is in `hugin-service/src/telemetry.rs`
- **No phone-home** -- when disabled, zero network requests are made to the telemetry backend

## What Is Collected

When enabled, Hugin sends anonymous usage events:

- **Feature usage counts** -- which tools are used (e.g., "scanner started", "intruder run") without any target information
- **Version and platform** -- Hugin version, OS, architecture
- **Session duration** -- how long the application was running
- **Error counts** -- crash/error type counts (no stack traces or context)

What is never collected:

- URLs, hostnames, or IP addresses
- Request/response content
- Scanner findings or vulnerability details
- Account IDs or license keys
- File paths or system information beyond OS/arch

## Enabling/Disabling

### Via CLI

```bash
# Check current status
hugin config telemetry

# Enable
hugin config telemetry on

# Disable
hugin config telemetry off
```

### Via Config File

Edit `~/.config/hugin/config.toml`:

```toml
[telemetry]
enabled = false
```

## Configuration

Full telemetry configuration options:

```toml
[telemetry]
# Master switch
enabled = false

# Backend endpoint
endpoint = "https://telemetry.hugin.nu/v1/events"

# How often to flush buffered events (seconds, minimum: 10)
flush_interval_secs = 300

# Maximum events to buffer before eager flush (1-10000)
max_batch_size = 100

# Extra key-value tags added to every event batch
[telemetry.tags]
# team = "security"
# environment = "lab"
```

## Data Flow

When enabled:

1. Usage events are buffered in memory
2. Every `flush_interval_secs` seconds (default: 5 minutes), or when the buffer reaches `max_batch_size`, events are sent as a single HTTP POST to the configured endpoint
3. If the endpoint is unreachable, events are silently dropped (no retry, no local persistence)
4. On clean shutdown, remaining buffered events are flushed

## Self-Hosted Telemetry

For teams that want usage analytics without sending data externally, point the endpoint to your own server:

```toml
[telemetry]
enabled = true
endpoint = "https://your-internal-server.example.com/v1/events"

[telemetry.tags]
team = "appsec"
```

The event format is a JSON array of event objects. See `hugin-service/src/telemetry.rs` for the exact schema.
