# Bug Bounty Platforms

Hugin integrates with the four major bug bounty platforms, allowing you to manage API credentials, sync program scopes, and pull program details directly into your workflow.

**Supported platforms:**

- **HackerOne** -- world's largest bug bounty platform
- **Bugcrowd** -- enterprise-focused bug bounty platform
- **YesWeHack** -- European bug bounty platform (GDPR-focused)
- **Intigriti** -- European bug bounty and VDP platform

Each platform is configured independently. You can enable one, all four, or any combination.

## Configuration

### UI (Settings > Platforms)

Navigate to **Settings > Platforms** in the Hugin desktop app. Each platform has its own card with:

- **API Key** -- your platform token (stored locally, never sent to Hugin servers)
- **Username** -- required for HackerOne (your H1 handle), not needed for other platforms
- **Base URL** -- pre-filled with the platform's default API endpoint, editable for enterprise/self-hosted instances
- **Enabled** -- toggle to activate/deactivate the integration

Click **Save** to persist credentials to `~/.hugin/platforms.json`. Use **Test Connection** to verify your API key works before starting work. **Remove** clears a platform's credentials entirely.

### MCP Tool

The `platforms` MCP tool exposes the same functionality for AI-driven workflows:

```
# List all configured platforms
platforms { action: "list" }

# Configure YesWeHack
platforms {
  action: "set",
  platform: "yeswehack",
  api_key: "your-pat-token"
}

# Configure HackerOne (requires username)
platforms {
  action: "set",
  platform: "hackerone",
  api_key: "your-api-key",
  username: "your-h1-handle"
}

# Test connectivity
platforms { action: "test", platform: "yeswehack" }

# Sync all programs you have access to
platforms { action: "sync_programs", platform: "yeswehack" }

# Get program details including scope
platforms { action: "get_program", platform: "yeswehack", program: "program-slug" }

# Remove credentials
platforms { action: "remove", platform: "bugcrowd" }
```

## Getting API Keys

### HackerOne

1. Go to [hackerone.com/settings/api_token](https://hackerone.com/settings/api_token)
2. Generate an API token
3. In Hugin, set both the **API Key** and your **Username** (your H1 handle)
4. Auth method: HTTP Basic Authentication (`username:api_key`)

### Bugcrowd

1. Go to [bugcrowd.com/settings/api-token](https://bugcrowd.com/settings/api-token)
2. Generate an API token
3. Auth method: `Authorization: Token <api_key>`

### YesWeHack

1. Log into [yeswehack.com](https://yeswehack.com)
2. Go to **MyYesWeHack** > **Create Token**
3. Set name, validity period, and scope
4. Save the token immediately -- it is only shown once
5. Auth method: `X-AUTH-TOKEN: <api_key>`

### Intigriti

1. Go to [app.intigriti.com/researcher/settings](https://app.intigriti.com/researcher/settings)
2. Generate an API token under the API section
3. Auth method: `Authorization: Bearer <api_key>`

## Syncing Programs

Once authenticated, use `sync_programs` to fetch all programs you have access to on a platform. This returns the full list with metadata (title, slug, bounty info, scope count).

Combined with the existing `project import_scope` action, you can pull a program's scope directly into a Hugin project:

```
# 1. Create a project for the target
project { action: "create", name: "Acme Corp", platform: "yeswehack", program_url: "https://yeswehack.com/programs/acme-corp" }

# 2. Import scope from the program page
project { action: "import_scope", id: "<project-id>", url: "https://yeswehack.com/programs/acme-corp" }
```

## Storage

All platform credentials are stored locally in `~/.hugin/platforms.json`. This file is never uploaded, synced, or transmitted. The format:

```json
{
  "platforms": {
    "yeswehack": {
      "enabled": true,
      "api_key": "ywh_...",
      "base_url": "https://api.yeswehack.com"
    },
    "hackerone": {
      "enabled": true,
      "api_key": "...",
      "username": "your-handle",
      "base_url": "https://api.hackerone.com"
    }
  }
}
```

When viewing credentials via the MCP `get` action, API keys are masked (first 4 and last 4 characters shown).

## Security Notes

- API keys are stored in plaintext on disk. Protect `~/.hugin/platforms.json` with appropriate file permissions.
- Keys are never sent to Hugin's servers -- all API calls go directly to the platform.
- The `test` action only fetches the first page of programs (minimal data) to verify authentication.
- Rate limiting is respected via standard HTTP timeouts. If a platform rate-limits you, retry after the indicated period.
