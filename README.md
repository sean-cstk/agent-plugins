# Agent Plugins

Public plugin examples and tooling for agent platforms, focused on practical workflows for real customer operations.

## Purpose

This repository is for plugin-oriented work that does not fit cleanly into generic `SKILL.md`-only distributions.

Initial focus areas:

- Claude Cowork plugin packaging and install UX
- Multi-account credential workflows
- Secure token handling patterns for agent tools
- Cross-platform compatibility notes (Claude, Codex, Copilot, and others)

## Security Principles

- Never paste live API tokens into chat prompts.
- Prefer environment files with strict permissions (`chmod 600`) or OS keychains.
- Use masked token output in logs and status messages.
- Prefer connector/MCP designs for long-term secret isolation.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
