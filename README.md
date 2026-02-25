# Compound Marketplace

[![Build Status](https://github.com/sglyon/compound-engineering-plugin/actions/workflows/ci.yml/badge.svg)](https://github.com/sglyon/compound-engineering-plugin/actions/workflows/ci.yml)
[![npm](https://img.shields.io/npm/v/@every-env/compound-plugin)](https://www.npmjs.com/package/@every-env/compound-plugin)

A Claude Code plugin marketplace featuring the **Compound Engineering Plugin** — tools that make each unit of engineering work easier than the last.

> **Fork notice:** This is a fork of [Kieran Klaassen's](https://github.com/kieranklaassen) original [compound-engineering-plugin](https://github.com/EveryInc/compound-engineering-plugin). The original plugin and its compounding engineering philosophy are entirely Kieran's work. This fork replaces the file-based todo tracking system (`todos/` directory with markdown files) with [Beads](https://github.com/steveyegge/beads), a git-backed issue tracker designed for AI coding agents.

## Claude Code Install

```bash
/plugin marketplace add https://github.com/sglyon/compound-engineering-plugin
/plugin install compound-engineering
```

## Other AI Coding Tools (experimental)

This repo includes a Bun/TypeScript CLI that converts Claude Code plugins to OpenCode, Codex, Factory Droid, Pi, Gemini CLI, GitHub Copilot, and Kiro CLI.

```bash
# convert to OpenCode format
bun run src/index.ts install ./plugins/compound-engineering --to opencode

# convert to Codex format
bun run src/index.ts install ./plugins/compound-engineering --to codex

# convert to Factory Droid format
bun run src/index.ts install ./plugins/compound-engineering --to droid

# convert to Pi format
bun run src/index.ts install ./plugins/compound-engineering --to pi

# convert to Gemini CLI format
bun run src/index.ts install ./plugins/compound-engineering --to gemini

# convert to GitHub Copilot format
bun run src/index.ts install ./plugins/compound-engineering --to copilot

# convert to Kiro CLI format
bun run src/index.ts install ./plugins/compound-engineering --to kiro
```

OpenCode output is written to `~/.config/opencode` by default. Commands are written as individual `.md` files. `opencode.json` (MCP servers) is deep-merged into any existing file — user keys are preserved.
Codex output is written to `~/.codex/prompts` and `~/.codex/skills`, with each Claude command converted into both a prompt and a skill. Generated Codex skill descriptions are truncated to 1024 characters (Codex limit).
Droid output is written to `~/.factory/` with commands, droids (agents), and skills.
Pi output is written to `~/.pi/agent/` with prompts, skills, extensions, and MCPorter config.
Gemini output is written to `.gemini/` with skills, commands (`.toml`), and `settings.json` (MCP servers).
Copilot output is written to `.github/` with agents (`.agent.md`), skills, and `copilot-mcp-config.json`.
Kiro output is written to `.kiro/` with custom agents, skills, steering files, and `mcp.json`.

All provider targets are experimental and may change as the formats evolve.

## Sync Personal Config

Sync your personal Claude Code config (`~/.claude/`) to other AI coding tools:

```bash
# Sync skills and MCP servers to OpenCode
bun run src/index.ts sync --target opencode

# Sync to Codex
bun run src/index.ts sync --target codex

# Sync to Pi
bun run src/index.ts sync --target pi

# Sync to Droid (skills only)
bun run src/index.ts sync --target droid

# Sync to GitHub Copilot (skills + MCP servers)
bun run src/index.ts sync --target copilot
```

This syncs:
- Personal skills from `~/.claude/skills/` (as symlinks)
- MCP servers from `~/.claude/settings.json`

Skills are symlinked (not copied) so changes in Claude Code are reflected immediately.

## Workflow

```
Plan → Work → Review → Compound → Repeat
```

| Command | Purpose |
|---------|---------|
| `/workflows:plan` | Turn feature ideas into detailed implementation plans |
| `/workflows:work` | Execute plans with worktrees and task tracking |
| `/workflows:review` | Multi-agent code review before merging |
| `/workflows:compound` | Document learnings to make future work easier |

Each cycle compounds: plans inform future plans, reviews catch more issues, patterns get documented.

## Philosophy

**Each unit of engineering work should make subsequent units easier—not harder.**

Traditional development accumulates technical debt. Every feature adds complexity. The codebase becomes harder to work with over time.

Compound engineering inverts this. 80% is in planning and review, 20% is in execution:
- Plan thoroughly before writing code
- Review to catch issues and capture learnings
- Codify knowledge so it's reusable
- Keep quality high so future changes are easy

## Learn More

- [Full component reference](plugins/compound-engineering/README.md) - all agents, commands, skills
- [Compound engineering: how Every codes with agents](https://every.to/chain-of-thought/compound-engineering-how-every-codes-with-agents)
- [The story behind compounding engineering](https://every.to/source-code/my-ai-had-already-fixed-the-code-before-i-saw-it)
