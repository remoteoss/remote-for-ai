# Remote plugin for Claude Code

This directory contains the Claude Code-specific manifest for the `remote-for-ai` plugin.

## Install

```bash
/plugin marketplace add remoteoss/remote-for-ai
/plugin install remote@remote-plugin-marketplace
```

After installing, restart Claude Code and verify with:

```bash
/help    # Should list /remote:* skills
/mcp     # Should show the remote MCP server
```

The plugin's MCP server uses OAuth 2.0 — the first time you call a Remote tool, Claude will prompt you to log in via your browser.

## Files

- [plugin.json](plugin.json) — plugin manifest (name, version, MCP server config).
- [marketplace.json](marketplace.json) — single-plugin marketplace descriptor for self-hosted installs.
