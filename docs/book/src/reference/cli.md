# CLI Reference

Hugin ships as a single binary. Run without arguments to launch the desktop GUI. Run with a subcommand for CLI mode.

```bash
hugin              # Launch GUI
hugin <command>    # CLI mode
```

Global flags:

- `-v`, `--verbose` -- Enable verbose logging (applies to all subcommands)

## Commands

### `hugin start`

Start the proxy daemon.

```
hugin start [OPTIONS]
```

Options:

- `-p`, `--port <PORT>` -- Proxy listen port (default: 8080)
- `--api-port <PORT>` -- API server port (default: 8081)
- `--bind <ADDRESS>` -- Bind address (default: 127.0.0.1)
- `--mcp` -- Also start the MCP server connected to the API

Example:

```bash
hugin start --port 8080 --api-port 8081 --mcp
```

### `hugin status`

Show proxy status including whether it is running, the listen address, and basic statistics.

```
hugin status
```

### `hugin ca`

Export the proxy CA certificate for browser/system trust store installation.

```
hugin ca [OPTIONS]
```

Options:

- `-o`, `--output <PATH>` -- Output path for the certificate file
- `--print` -- Print certificate to stdout (PEM format)

Example:

```bash
hugin ca -o ~/Hugin-CA.pem
hugin ca --print | sudo tee /usr/local/share/ca-certificates/hugin.crt
```

### `hugin flows`

List captured HTTP flows.

```
hugin flows [OPTIONS]
```

Options:

- `-m`, `--method <METHOD>` -- Filter by HTTP method (GET, POST, etc.)
- `--host <HOST>` -- Filter by hostname
- `--flagged` -- Show only flagged flows
- `-l`, `--limit <N>` -- Maximum results (default: 20)
- `-f`, `--format <FORMAT>` -- Output format: `table` or `json` (default: table)

Example:

```bash
hugin flows --host example.com --method POST --format json
hugin flows --flagged --limit 50
```

### `hugin flow`

Show details of a specific flow by ID.

```
hugin flow <ID>
```

Example:

```bash
hugin flow a3f2e1d0-1234-5678-9abc-def012345678
```

### `hugin mcp`

Start the embedded MCP server on stdio for integration with Claude Code and Claude Desktop. The MCP server starts its own proxy and API internally.

```
hugin mcp
```

This is the command specified in MCP client configurations:

```json
{
  "mcpServers": {
    "hugin": {
      "command": "hugin",
      "args": ["mcp"]
    }
  }
}
```

### `hugin init`

Generate a default configuration file at `~/.config/hugin/config.toml`.

```
hugin init
```

### `hugin setup`

First-run setup wizard. Configures CA certificate installation, browser proxy settings, and basic options.

```
hugin setup [OPTIONS]
```

Options:

- `--headless` -- Non-interactive mode for CI/headless environments

### `hugin serve`

Start a headless server for team collaboration. Binds to `0.0.0.0` by default for remote access. No GUI is launched.

```
hugin serve [OPTIONS]
```

Options:

- `-p`, `--port <PORT>` -- Proxy listen port (default: 8080)
- `--api-port <PORT>` -- API server port (default: 8081)
- `--bind <ADDRESS>` -- Bind address (default: 0.0.0.0)
- `--no-auth` -- Disable token authentication (DANGEROUS -- only use on trusted networks)

Example:

```bash
# Start headless server for team
hugin serve --port 8080 --api-port 8081

# On a trusted local network only
hugin serve --no-auth
```

### `hugin update`

Check for updates or update Hugin in-place.

```
hugin update [OPTIONS]
```

Options:

- `--check` -- Only check for updates, do not install

### `hugin verify`

Verify the Ed25519 signature of a downloaded Hugin release file.

```
hugin verify <FILE> [OPTIONS]
```

Options:

- `-s`, `--sig <PATH>` -- Path to the `.sig` file (defaults to `<file>.sig`)

Example:

```bash
hugin verify hugin-linux-amd64.tar.gz
hugin verify hugin-linux-amd64.tar.gz --sig custom-path.sig
```

### `hugin account`

Manage your Hugin Pro license account.

#### `hugin account set`

Activate Pro license with your account ID.

```
hugin account set <ID>
```

The account ID format is `HGN-XXXXXXXX-XXXXXXXX-XXXXXXXX` (from your purchase confirmation).

#### `hugin account show`

Show current account and license status.

```
hugin account show
```

#### `hugin account clear`

Deactivate and remove the stored account ID.

```
hugin account clear
```

### `hugin scanner`

Manage Synaps WASM scanner modules.

#### `hugin scanner update`

Sync modules from the community catalog.

```
hugin scanner update [OPTIONS]
```

Options:

- `--catalog-url <URL>` -- Custom catalog URL (defaults to synaps-community GitHub)
- `--base-url <URL>` -- Base URL for downloading WASM binaries
- `--dry-run` -- Only check for updates, do not download

#### `hugin scanner install`

Install a specific module by ID.

```
hugin scanner install <ID> [OPTIONS]
```

Options:

- `--catalog-url <URL>` -- Custom catalog URL
- `--base-url <URL>` -- Base URL for WASM binaries

#### `hugin scanner remove`

Remove an installed module.

```
hugin scanner remove <ID>
```

#### `hugin scanner list`

List installed modules.

```
hugin scanner list
```

### `hugin config`

Manage proxy configuration.

#### `hugin config telemetry`

Enable, disable, or check telemetry status.

```
hugin config telemetry [ACTION]
```

Action is `on` to enable, `off` to disable, or omit to show current status.

#### `hugin config show`

Show current configuration summary.

```
hugin config show
```

### `hugin plugin`

Manage Lua plugins.

#### `hugin plugin install`

Install a plugin from a Git repository URL.

```
hugin plugin install <URL>
```

Example:

```bash
hugin plugin install https://github.com/user/hugin-plugin-name
```

#### `hugin plugin remove`

Remove an installed plugin by directory name.

```
hugin plugin remove <NAME>
```

#### `hugin plugin list`

List installed plugins.

```
hugin plugin list
```

### `hugin token`

Manage access tokens for team collaboration.

#### `hugin token create`

Create a new access token.

```
hugin token create [OPTIONS]
```

Options:

- `-l`, `--label <LABEL>` -- Optional label for the token (e.g., team member name)

#### `hugin token list`

List all access tokens.

```
hugin token list
```

#### `hugin token revoke`

Revoke an access token.

```
hugin token revoke <TOKEN>
```

The full `hgn_*` token string is required.
