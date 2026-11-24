# homelab-memory-claude-code-mod

A Claude Code mod that copies Claude Code's memory into self-hosted memory MCP servers, alongside the harness's own stores:

- a **markdown vault** MCP server for durable notes, facts and procedures
- **mindmeld**, a search index over past agent conversations

Claude Code's own memory keeps working as normal, and the vault wins. Each memory write runs, then gets copied to the vault. Each memory read first refreshes the local file from the vault, and backfills whichever side is behind. If the vault is down, everything runs natively and a catch-up scan syncs later.

The point is vendor independence. The memory lives in stores the owner controls, versions and backs up, so any harness or model (Claude Code, Codex, a local model) reads and writes the same record.

## Status

Research only, as of 2026-11-23. No mod code yet. The research covers Claude Code **2.1.331**.

| Doc | What it answers |
|---|---|
| [docs/native-memory.md](docs/native-memory.md) | Which memory systems Claude Code has, and how to turn each off |
| [docs/hook-points.md](docs/hook-points.md) | Where a mod can intercept them, by tool name, with code sketches |
| [docs/sync.md](docs/sync.md) | The sync design: writes mirror to the vault, reads prefer the vault and backfill |
| [docs/patchers.md](docs/patchers.md) | Third-party patchers (ccmod, tweakcc) and why this mod doesn't use them |
| [docs/open-questions.md](docs/open-questions.md) | What is still unknown and the decisions not yet made |

## Prior art

- [redaphid/draw-mod](https://github.com/redaphid/draw-mod) is a Claude Code mod built on the same plugin hooks-module API. This mod follows its layout.
- Classic `settings.json` hooks already handle part of this job on one machine: a `PreToolUse` guard denies writes into memory directories, and a `SessionStart` hook injects a memory contract. The mod would replace them with one installable plugin. [docs/hook-points.md](docs/hook-points.md#classic-hooks-vs-the-mod-api) says why the mod API is stronger.
