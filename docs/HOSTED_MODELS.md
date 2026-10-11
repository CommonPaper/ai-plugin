# Hosted models: utilization guide and merge strategy

**TL;DR: The Common Paper plugin works with any MCP-capable hosted model through the remote MCP server `https://api.commonpaper.com/mcp` (Streamable HTTP, OAuth with dynamic client registration). The hosted server is the single canonical surface; local stdio builds are for development only.**

This document defines how the plugin reaches the broadest set of hosted models, and the strategy the repository follows when a new host surface appears.

## Compatibility matrix

| Host surface | Config surface | Transport | Auth |
|---|---|---|---|
| ChatGPT (connectors) | Chat settings -> Connectors / developer MCP config | Streamable HTTP | OAuth (dynamic client registration) |
| OpenAI Codex CLI | `mcp.json` / config `mcp_servers` entry | Streamable HTTP | OAuth |
| Claude Code / Cowork | Plugin install (this repo) or `.mcp.json` | Streamable HTTP | OAuth |
| Cursor | Plugin install (this repo) or `.mcp.json` | Streamable HTTP | OAuth |
| VS Code / GitHub Copilot | `.vscode/mcp.json` | Streamable HTTP | OAuth |
| Gemini CLI | `gemini-extension.json` (plugin) or `settings.json` `mcpServers` | Streamable HTTP | OAuth |
| Grok Build | `.grok-plugin/marketplace.json` (plugin) | Streamable HTTP | OAuth |
| Any other MCP client | `mcp.json` / `mcpServers` entry | Streamable HTTP | OAuth |

All surfaces point at the same remote: `https://api.commonpaper.com/mcp`. There is exactly one server, one protocol (MCP Streamable HTTP), and one auth path (Common Paper OAuth, no API keys).

## Config snippets

### OpenAI Codex CLI

Add to `mcp.json` (or the `mcp_servers` config block):

```json
{
  "mcpServers": {
    "common-paper": {
      "url": "https://api.commonpaper.com/mcp"
    }
  }
}
```

### ChatGPT connectors

In ChatGPT, open Settings -> Connectors -> Advanced (developer mode), create a MCP connector, and set the server URL to `https://api.commonpaper.com/mcp`. Sign in with your Common Paper account when prompted.

### Any MCP client

```json
{
  "mcpServers": {
    "common-paper": {
      "type": "http",
      "url": "https://api.commonpaper.com/mcp"
    }
  }
}
```

## Merge strategy for new hosted-model surfaces

The repository follows four rules when a new host surface appears (a new AI host, a new plugin format, a new marketplace):

1. **Hosted remote first.** Every new surface points at `https://api.commonpaper.com/mcp`. A surface that cannot speak MCP Streamable HTTP with OAuth is not supported by this repo; the REST API skill fallback covers the gap.
2. **Config file per host, server facts once.** Each host gets its own small config or manifest (as `.cursor-plugin/`, `.claude-plugin/`, `.grok-plugin/`, `gemini-extension.json` already do). Server URL, transport, and auth live in exactly one place per file, mirroring `registry/server.json`.
3. **Skill surfaces travel with the connector.** When a host supports skills or instructions, ship the existing `skills/commonpaper/` content with it; do not fork the skill text per host. One skill, many surfaces.
4. **No local stdio builds in end-user paths.** Local stdio server builds are development tools only. End-user docs, manifests, and marketplaces reference the hosted remote only. When a local build and the hosted remote diverge, the hosted remote is the source of truth, and the local build follows.

## Versioning

Manifests in this repo (`plugin.json`, `gemini-extension.json`, `registry/server.json`) version together in lockstep. A change to any host surface or to the merge strategy bumps the version in all manifests in the same commit.
