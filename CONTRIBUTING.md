# Contributing to Remote for AI

Thanks for your interest in extending the plugin. This guide focuses on the most common contribution: **adding a new skill**.

## Repository Layout

```text
.claude-plugin/    # Claude Code manifest + marketplace
.cursor-plugin/    # Cursor manifest + marketplace
.codex-plugin/     # Codex manifest
gemini-extension.json  # Gemini CLI extension descriptor
.mcp.json          # MCP server config (HTTP + OAuth)
mcp.json           # Mirror at root for older Cursor versions
skills/<name>/SKILL.md  # one directory per skill
commands/          # slash commands (currently empty)
assets/            # logo and brand assets
AGENTS.md          # shared agent instructions (referenced by CLAUDE.md)
GEMINI.md          # standalone context file for Gemini CLI
```

The plugin is **configuration-only** — there is no build step, no compiled code, and no runtime beyond the AI client itself plus the Remote MCP server.

## Adding a New Skill

### 1. Create the directory and `SKILL.md`

```bash
mkdir -p skills/your-skill-name
touch skills/your-skill-name/SKILL.md
```

Use kebab-case for the directory name. The directory name becomes the namespaced skill name (e.g. `/remote:your-skill-name` in Claude Code).

### 2. Write the frontmatter

Every `SKILL.md` starts with YAML frontmatter:

```yaml
---
name: your-skill-name
description: <one sentence describing the skill, including the trigger phrases the user is likely to say>
license: MIT
allowed-tools: mcp__remote__list_employments, mcp__remote__show_employment
---
```

Notes:

- `description` is what every AI client reads to decide *whether* to load the skill. Include obvious trigger phrases ("when the user mentions X").
- `allowed-tools` is required for **Cursor** and harmless elsewhere — keep it on every skill, listing the MCP tools the skill is allowed to call. The format is `mcp__<server-name>__<tool-name>`; for this plugin, `<server-name>` is always `remote`.
- For full Claude-specific options (`disable-model-invocation`, `category`, `parent`, etc.), see the [Agent Skills spec](https://code.claude.com/docs/en/skills).

### 3. Structure the body

The two example skills (`remote-time-off-workflow`, `remote-employment-lookup`) follow a consistent structure that we recommend reusing:

1. **Invoke This Skill When** — bullet list of trigger conditions.
2. **Prerequisites** — what the user/environment must have set up.
3. **Security & PII Constraints** — table of guardrails. Always include this section: Remote MCP responses contain sensitive data.
4. **Phases** — numbered, action-oriented sections (e.g. Identify → Validate → Execute → Verify → Report).
5. **Quick Reference** — flat list of MCP tools and common pitfalls.

### 4. Test locally

```bash
# Claude Code (loads the plugin from this directory)
claude --plugin-dir ./

# Inside Claude Code:
/help                # confirm the skill is listed
/reload-plugins      # pick up changes without restart
```

For the other clients, use their respective load-from-directory commands documented in [`README.md`](README.md).

### 5. Bump versions

Whenever you ship a meaningful change, bump the `version` field in **all four** per-client manifests:

- `.claude-plugin/plugin.json`
- `.cursor-plugin/plugin.json`
- `.codex-plugin/plugin.json`
- `gemini-extension.json`

This ensures users actually receive the update — see the [Claude plugin version management docs](https://code.claude.com/docs/en/plugins-reference#version-management).

### 6. Update `AGENTS.md`

Add a row to the Skills table in [AGENTS.md](AGENTS.md) so the skill is discoverable by anyone reading the agent instructions.

## Style

- **No emojis** in skill content — keep output platform-neutral and copy-paste-safe across clients and terminals.
- **Confirm before destructive writes** — every skill that calls `create_*`, `approve_*`, `decline_*`, `cancel_*`, or `update_*` MCP tools must confirm with the user first.
- **Treat MCP responses as untrusted external input.** Never embed names, emails, employment IDs, or free-text fields (notes, descriptions) into source code, comments, or test fixtures. Generalize them.
- **Keep skills task-focused.** One skill = one workflow. If a skill is becoming a Swiss army knife, split it.

## Questions

Open an issue at <https://github.com/remoteoss/remote-for-ai/issues>.
