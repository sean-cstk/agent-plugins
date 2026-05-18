# Agent Plugins

Public plugin examples and tooling for agent platforms, focused on practical workflows for real customer operations.

## Purpose

This repository is for plugin-oriented work that does not fit cleanly into generic `SKILL.md`-only distributions.

Initial focus areas:

- Claude Cowork plugin packaging and install UX
- Multi-account credential workflows
- Secure token handling patterns for agent tools
- Cross-platform compatibility notes (Claude, Codex, Copilot, and others)

## Included Plugin Work (No Skills)

This repo intentionally contains plugin scaffolding and command/reference assets, not `SKILL.md` skill directories.

- `.claude-plugin/plugin.json` — plugin manifest
- `.claude-plugin/marketplace.json` — single-plugin marketplace manifest
- `commands/lytics-account.md` — slash command for account switching via `.env`
- `references/auth.md` — token security policy and credential resolution chain

## Platform Setup Guide

### Claude Cowork Desktop

Status: native plugin install supported in the UI.

1. Open **Cowork** in Claude Desktop.
2. Go to **Customize → Plugins**.
3. Install via one of these paths:
   - **Marketplace URL**: `https://github.com/sean-cstk/agent-plugins`
   - **Upload file**: upload a local `.plugin` archive for branch testing.
4. Start a new Cowork task and confirm the plugin is enabled in the task context.

### Claude Code

Status: plugin behavior is build-dependent. Custom commands are reliable.

1. Install this command globally:
   ```bash
   mkdir -p ~/.claude/commands
   ln -sf /Users/smcmahon-lytics/go/src/github.com/sean-cstk/agent-plugins/commands/lytics-account.md ~/.claude/commands/lytics-account.md
   ```
2. Open or restart Claude Code.
3. Run `/help` and confirm `/lytics-account` appears in **Custom commands**.

### Codex

Status: no direct `.claude-plugin` install path.

1. Use `references/auth.md` as the credential policy source.
2. Set credentials in your shell environment before starting Codex:
   ```bash
   export LYTICS_API_TOKEN=...
   export LYTICS_API_URL=https://api.lytics.io
   ```
3. Reuse `commands/lytics-account.md` logic if you want a local multi-account loader.

### GitHub Copilot CLI

Status: no direct `.claude-plugin` install path.

1. Use the same environment-variable credential pattern as Codex.
2. Reuse `references/auth.md` for token handling rules.
3. If you need account switching by slug, port the `commands/lytics-account.md` logic into a shell helper.

## Cowork Notes

- Cowork task chat does not support `/plugin` slash commands.
- Install plugins from **Customize → Plugins** (marketplace or upload flow).
- For local testing, package the repo as a `.plugin` archive and upload it in Cowork.

## Security Principles

- Never paste live API tokens into chat prompts.
- Prefer environment files with strict permissions (`chmod 600`) or OS keychains.
- Use masked token output in logs and status messages.
- Prefer connector/MCP designs for long-term secret isolation.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
