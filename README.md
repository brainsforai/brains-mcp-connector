# Brains MCP Connector

This repository documents the [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) server that powers [Brains](https://usebrains.app) — a personal wiki and memory layer for AI assistants.

It is published as a **transparency document**: you can inspect exactly which tools are exposed, what parameters they accept, how authentication works, and what data flows through the connector. The server backend is proprietary; what you see here is the full public API surface.

---

## What is Brains?

Brains gives Claude, ChatGPT, and other MCP-compatible AI assistants persistent, structured memory backed by a private wiki you control. Instead of each conversation starting blank, your assistant can read and write to your wiki across sessions.

---

## How to connect

### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "brains": {
      "type": "http",
      "url": "https://usebrains.app/mcp"
    }
  }
}
```

Claude will prompt you to authorize via OAuth the first time.

### Claude Code (CLI)

```bash
claude mcp add brains --transport http https://usebrains.app/mcp
```

### Manual MCP client

Connect via StreamableHTTP to `https://usebrains.app/mcp` with an `Authorization: Bearer <token>` header. Obtain a token via the OAuth 2.0 flow described below.

---

## Authentication

Brains uses **OAuth 2.0** (RFC 6749) with PKCE.

| Endpoint | URL |
|----------|-----|
| Authorization | `https://usebrains.app/oauth/authorize` |
| Token | `https://usebrains.app/api/oauth/token` |

**Scopes:**

| Scope | Access |
|-------|--------|
| `wiki.read` | List, read, and search wiki pages |
| `wiki.write` | All read operations plus create, edit, move, and delete pages |

MCP connections require `wiki.write`. A `wiki.read`-only scope is reserved for future read-only integrations.

**Token lifetimes:**
- Access token: 1 hour
- Refresh token: 30 days

Sessions are maintained server-side by session ID and survive server restarts via transparent reconnect.

---

## Transport

Brains supports two MCP transports:

| Transport | Endpoint | Use case |
|-----------|----------|----------|
| StreamableHTTP | `https://usebrains.app/mcp` | Production / cloud use |
| stdio | n/a (self-host only) | Local development |

---

## Tools

All tools are scoped to the authenticated user's wiki. Cross-user data access is not possible at the storage or API layer.

The complete input schemas for every tool are in [`tools.json`](tools.json).

### Core wiki operations

| Tool | Access | Description |
|------|--------|-------------|
| `list_pages` | R | List pages with optional tag/status filters |
| `read_page` | R | Read a page by filename (returns content, frontmatter, revision) |
| `search_pages` | R | Full-text search across wiki pages |
| `create_page` | W | Create a new page |
| `create_pages` | W | Batch create pages in one call |
| `update_page` | W | Full rewrite of a page (requires revision token from read_page) |
| `patch_page` | W | Surgical edits: replace text, insert lines, append sections |
| `move_page` | W | Rename or archive a page |
| `delete_page` | W | Hard delete a single page (dry-run by default) |
| `batch_delete_pages` | W | Hard delete up to 50 pages (dry-run by default) |

### Session bundles

| Tool | Access | Description |
|------|--------|-------------|
| `pull_session_bundle` | R | Download all project pages as a single text bundle |
| `push_session_bundle` | W | Write back pages modified during a session |

Bundles allow the AI to load all relevant pages at once (2.6–4.2× faster than reading one by one) and push back changes in a single call.

### Bulk import/export

| Tool | Access | Description |
|------|--------|-------------|
| `export_wiki` | R | Export all pages to a ZIP archive (returned inline as base64) |
| `import_wiki` | W | Import pages from a ZIP archive (additive or replace modes) |
| `purge_wiki` | W | Permanently delete all pages (dry-run by default) |

### Project setup

| Tool | Access | Description |
|------|--------|-------------|
| `init_project` | W | First-time project scaffolding — creates canonical page structure |
| `generate_claude_instructions` | R | Refresh the Claude system-prompt block for a project |
| `generate_gpt_instructions` | R | Refresh the ChatGPT system-prompt block for a project |

### Diagnostics

| Tool | Access | Description |
|------|--------|-------------|
| `health_check` | R | Verify storage and index connectivity |
| `audit_index` | R/W | Find and repair index/storage drift |
| `append_log` | W | Write a structured event to the wiki log |

### Link graph

| Tool | Access | Description |
|------|--------|-------------|
| `extract_links` | W | Backfill the link graph from page content |
| `get_backlinks` | R | Pages that link to a given page |
| `get_related` | R | Pages sharing outbound link targets |
| `suggest_links` | R | Find pages missing cross-references |

### Webhooks

| Tool | Access | Description |
|------|--------|-------------|
| `set_webhook` | W | Create or update an outbound webhook rule |
| `list_webhooks` | R | List your webhook rules |
| `remove_webhook` | W | Delete a webhook rule |
| `add_webhook_pattern` | W | Append a glob or regex pattern to a rule |

Webhooks fire a signed POST to a URL of your choosing whenever a matching page is created or updated. Patterns use glob syntax (e.g. `projects/myapp/*.md`) or `/regex/` syntax.

---

## Data handling

### What Brains reads and writes

Every MCP tool call is scoped to the authenticated user's wiki namespace. No tool can read another user's pages.

**Data the connector reads:**
- Your wiki page names, frontmatter, and body content (on `read_page`, `list_pages`, `search_pages`, `pull_session_bundle`, `export_wiki`)

**Data the connector writes:**
- Your wiki page content (on `create_page`, `update_page`, `patch_page`, `push_session_bundle`, `import_wiki`)
- Webhook rule configurations (on `set_webhook`, `add_webhook_pattern`)
- Structured log entries (on `append_log`)

### What Brains stores

- Your wiki pages (markdown files with YAML frontmatter)
- OAuth access and refresh tokens (to persist sessions across restarts)
- Structured usage logs (request timing, error events — not conversation content)
- Webhook rule configurations

### What Brains does NOT store

- Conversation content from your AI assistant (prompts, replies, context windows)
- Data from outside your wiki
- Any data from other users' wikis

### Third-party sharing

None. See the [privacy policy](https://usebrains.app/privacy).

### Data retention

Until account deletion. See [privacy policy](https://usebrains.app/privacy).

---

## Tool schemas

[`tools.json`](tools.json) contains the complete JSON Schema for every tool's input parameters, generated directly from the server source. These schemas are what the MCP server returns in response to a `tools/list` request.

---

## Architecture overview

```
AI assistant (Claude, ChatGPT, etc.)
       │
       │  MCP / JSON-RPC 2.0
       │  POST https://usebrains.app/mcp
       │  Authorization: Bearer <oauth-token>
       ▼
┌─────────────────────────────────────────┐
│           Brains MCP Server             │
│                                         │
│  ┌─────────────┐   ┌─────────────────┐  │
│  │  OAuth 2.0  │   │  Tool handlers  │  │
│  │  (token     │   │  (30 tools —    │  │
│  │   verify)   │   │   see above)    │  │
│  └─────────────┘   └────────┬────────┘  │
│                             │           │
└─────────────────────────────┼───────────┘
                              │
                    ┌─────────▼─────────┐
                    │   Wiki storage    │
                    │  (user-scoped,    │
                    │   isolated)       │
                    └───────────────────┘
```

The storage backend, wiki layer, and auth systems are proprietary and not included in this repository. This repository documents only the public MCP protocol surface.

---

## MCP manifest

See [`manifest.json`](manifest.json) for the full server manifest (transports, OAuth endpoints, scopes, data collection declaration).

---

## License

MIT
