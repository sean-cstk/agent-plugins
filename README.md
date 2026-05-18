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
