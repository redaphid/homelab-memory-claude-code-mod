# Sync design: writes mirror, reads prefer the vault, reads backfill

The requirement grew in three steps, all on 2026-11-23:

1. "We will want to 'fall back' to default behavior, and migrate on fallback."
2. "maybe we intercept the tool call and add the memory to my markdown vault, then let it continue default behavior."
3. "Reads should prioritize my self-hosted one, and reads should backfill."

So Claude Code's native memory keeps working, and the vault is the copy that wins:

- **Writes mirror.** The native write runs, then the mod copies the result to the vault.
- **Reads prefer the vault.** Before a native read runs, the mod refreshes the local file from the vault, so the model reads the vault's version through Claude Code's ordinary tools.
- **Reads backfill.** Each read repairs whichever side is behind. The vault's version fills a stale or missing native file, and a native file the vault lacks is copied up.
- **Vault down:** everything runs natively, unchanged, and a catch-up scan syncs later.

```
 READ   model ── Read(memory file) ──▶ mod: sync this file with the vault ──▶ next(e): native Read
                                         vault newer   → write it to the native file (backfill native)
                                         native only   → copy it to the vault        (backfill vault)
                                         vault down    → skip; native as is

 WRITE  model ── Write/Edit(memory file) ──▶ next(e): native write ──▶ on success: copy file to vault
                                                                        vault down → queue for catch-up
```

## Why refresh the local file instead of answering from the vault

A `tool.call` hook could return the vault's text as `{ result }` in place of the native Read. But Read's result has its own schema (line numbers, file type, truncation), and a hand-made one would drift from it. Writing the vault's text to the native file with `$.fs.write` and then calling `next(e)` gets the same outcome through the real tool. As a bonus, the native copy stays fresh for whenever the vault is down.

## Who wins: three-way comparison on a per-file sync record

`$.store` keeps one record per file: the content hash at the last successful sync. When reading or scanning, the mod compares three things: the native file, the vault copy, and that record.

| Native vs. record | Vault vs. record | Meaning | Action |
|---|---|---|---|
| same | same | in sync | nothing |
| same | changed | vault edited (by you, another harness or another machine) | **vault wins**: write it to native |
| changed | same | native written while the vault was down, or the copy failed | push native to the vault |
| changed | changed | both edited since the last sync | **conflict**: vault wins on the native side; keep native's version in the vault as `<file>.conflict-<date>.md` |
| missing | exists | a note the vault has, native lacks | write it to native |
| exists | missing | never synced | copy it to the vault |
| missing | missing, but the record exists | deleted on both sides | drop the record |

The record is what keeps "reads prefer the vault" safe. Without it, a read right after an outage would let the vault's older copy overwrite the newer native write that hadn't been copied up yet.

## Hooks

| Native operation | Hook | Mod does |
|---|---|---|
| `Read` of a memory file | `tool.call` `{ tool: 'Read' }`, path-matched | Sync that one file, then `next(e)` |
| `Glob` / `Grep` / `LS` over a memory dir | `tool.call`, path-matched | Sync the directory (list the vault folder, pull missing or newer files), then `next(e)` |
| `MEMORY.md` loaded at session start | `session.start` (awaited before the first prompt) | Full sync of this project's memory dir |
| `Write` / `Edit` / `NotebookEdit` into a memory dir | `tool.call`, path-matched | `next(e)`, then copy on success and update the record |
| `memory_read` / `memory_write` (surfaces that have them) | `tool.call` by name | Same pattern; native backfill through `$.tool.call('memory_write', ...)` with the version token |
| Subagent memory dirs | Same tools, `agent-memory*` paths | Same |

Every sync step runs under a short timeout and its `.catch` falls through to `next(e)`. A slow or dead vault costs a read nothing but freshness.

## Catch-up scan

It runs at `session.start`, after a failed copy, and on a timer while anything is queued. It applies the table above to every file in this project's native memory dir and the matching vault folder. The first run is the one-time migration, which copies everything already in native memory up to the vault.

## Where copies live

`claude-memory/<project>/<file>` in the vault, one folder per Claude Code project, mirroring `~/.claude/projects/<project>/memory/`. Agent memory goes under `claude-memory/agents/<name>/`. Notes elsewhere in the vault (`kb/facts` and so on) are read through the vault's own MCP tools, not through this sync.

## Consequences

- **Native memory turns back on.** `autoMemoryEnabled: false`, `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` and the `PreToolUse` memory-dir guard all go when the mod ships.
- **Sync is two-way now.** Reads backfill native, so vault edits reach Claude Code at its next read or session start.
- **Native deletes** don't delete from the vault. Removing the file from the vault side is the way to delete a memory everywhere.

## To verify on a loaded mod

- That `session.start`'s sync lands before Claude Code reads `MEMORY.md` into context. If it doesn't, `prompt.context` or `prompt.section` can put the vault's `MEMORY.md` in directly.
- Whether Claude Code reads topic files with the `Read` tool or some internal path that `tool.call` doesn't see.
- The cost of a vault round trip per memory read, against the timeout.

## Set aside: routing to the vault, with a fallback mode

The first version had the mod answer memory calls from the vault (`{ result }` in place of the tool), switch to native when the vault was down, and migrate when it came back. That needs mode tracking, prompt-section swaps that fight the prompt cache, and results faked to each tool's schema. Mirror-plus-backfill gets the same outcome through the real tools.
