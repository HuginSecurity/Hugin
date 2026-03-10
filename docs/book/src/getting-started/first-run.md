# First Run

## Starting Hugin

Run `hugin` with no arguments to launch the desktop GUI:

```bash
hugin
```

The proxy starts automatically on `127.0.0.1:8080` and the API server on `127.0.0.1:8081`. On first run, Hugin generates a CA certificate and creates an empty project database.

### CLI Mode

If you prefer to run Hugin without the GUI (headless servers, SSH sessions, CI pipelines):

```bash
# Start proxy + API in the foreground
hugin start

# Custom ports
hugin start --port 9090 --api-port 9091

# Start with MCP server for Claude integration
hugin start --mcp

# Verbose logging
hugin start -v
```

Press `Ctrl+C` to stop the proxy.

### Headless Server Mode

For remote/team access, use `serve` which binds to `0.0.0.0` by default:

```bash
hugin serve
```

This requires token authentication. Create tokens with `hugin token create`.

## Setup Wizard

On first launch, Hugin runs a setup wizard that walks through proxy port selection, CA certificate installation, and optional Pro license activation. You can re-run it any time:

```bash
hugin setup
```

For non-interactive environments:

```bash
hugin setup --headless
```

## Data Directory

All Hugin data lives under `~/.config/hugin/`:

```
~/.config/hugin/
  config.toml              Configuration file
  Hugin-Proxy-CA.pem       CA certificate (install in your browser/OS)
  Hugin-Proxy-CA-key.pem   CA private key (keep private)
  hugin.db                 SQLite database (flows, findings, projects)
  modules/                 Synaps scanner modules (WASM)
  extensions/              Lua plugin scripts
```

## Generate a Default Config

If you want to customize settings before starting:

```bash
hugin init
```

This writes a default `config.toml` to `~/.config/hugin/config.toml`. See [Configuration](./configuration.md) for details.

## Export the CA Certificate

To export the CA certificate to a specific location:

```bash
hugin ca --output ~/Desktop/hugin-ca.pem
```

Or print it to stdout:

```bash
hugin ca --print
```

The certificate is also downloadable from the API at `http://127.0.0.1:8081/api/ca.pem` while Hugin is running.
