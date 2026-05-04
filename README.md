# Remote for AI

Official Remote plugin for AI coding assistants. Access Remote's global employment platform — employments, payroll, contractors, time off, expenses, and timesheets — directly from your AI coding tool.

Supports **Claude Code**, **Cursor**, **Codex**, and **Gemini CLI**.

## What You Can Do

The plugin gives your AI assistant the tools and context to act on your Remote workspace. Anything available through the Remote API — employments, time off, expenses, timesheets, payroll, contractors, leave policies, and more — can be driven from natural-language prompts in your editor. The plugin handles authentication, exposes Remote MCP tools, and ships task-specific skills that encode best practices, validation, and PII guardrails so the assistant stays accurate and safe.

## Installation

### Claude Code

```bash
/plugin marketplace add remoteoss/remote-for-ai
/plugin install remote@remote-plugin-marketplace
```

Restart Claude Code, then verify with:

```bash
/help    # Should list /remote:* skills
/mcp     # Should show the `remote` MCP server
```

The first time a Remote tool is invoked, Claude will prompt you to authenticate via OAuth in your browser.

### Cursor

Search for **Remote** in `Cursor Settings > Plugins` and install. Or, for development:

```bash
git clone https://github.com/remoteoss/remote-for-ai
ln -s "$PWD/remote-for-ai" ~/.cursor/plugins/local/remote
```

### Codex

```bash
codex plugin marketplace add remoteoss/remote-for-ai
```

Then run `/plugins` from inside Codex, select **Remote**, and install.

### Gemini CLI

```bash
gemini extensions install https://github.com/remoteoss/remote-for-ai
```

This bridges the HTTP MCP endpoint via `npx mcp-remote@latest`, since Gemini CLI currently only supports stdio MCP servers.

### From Source (any client)

```bash
git clone https://github.com/remoteoss/remote-for-ai

# Claude Code
claude --plugin-dir ./remote-for-ai

# Cursor
ln -s "$PWD/remote-for-ai" ~/.cursor/plugins/local/remote

# Codex
codex --plugin-dir ./remote-for-ai

# Gemini CLI
gemini extensions install ./remote-for-ai
```

## Skills

The plugin ships a growing library of task-specific skills under [`skills/`](skills/). Each skill encodes a single workflow with the right Remote MCP tools, validation steps, and PII handling rules baked in.

You don't need to know skill names: your AI client reads their descriptions and loads the right one based on your prompt. Browse the [`skills/`](skills/) directory to see what's currently available, or run `/help` from Claude Code to list them. To add a new skill, see [CONTRIBUTING.md](CONTRIBUTING.md).

## MCP Server

The plugin configures the Remote MCP server (HTTP transport, OAuth 2.0) automatically on install. No API keys to copy-paste; first-call OAuth handles authentication.

## Prerequisites

- A [Remote account](https://remote.com).
- One of: Claude Code, Cursor, Codex, or Gemini CLI installed.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to add new skills.

## License

MIT
