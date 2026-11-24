# Fallback and migration

Requirement, 2026-11-23: when the memory MCP servers can't be reached, Claude Code **falls back to its default memory behaviour**. Whatever it writes natively during the fallback is **migrated into the vault** once the vault is back.

So the mod has two modes, chosen by vault health, plus a migration step between them:

```
          vault reachable                         vault unreachable
   ┌──────────────────────────┐            ┌──────────────────────────┐
   │ ROUTED                   │  vault     │ FALLBACK                 │
   │ memory section -> vault  │  fails     │ native memory section    │
   │ memory_* -> vault        │ ─────────▶ │ native memory_* tools    │
   │ memory-dir writes denied │            │ native memory-dir writes │
   └──────────────────────────┘ ◀───────── └──────────────────────────┘
                                 vault back:
                                 MIGRATE native writes into the vault
```

## Consequence: native memory has to stay on

Fallback needs native memory switched on underneath the mod. Today it is off, and these settings have to go once the mod ships:

- `autoMemoryEnabled: false` and `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`. With either one set, there's nothing to fall back to.
- the classic `PreToolUse` memory guard that denies writes into memory directories. In fallback mode it would block the fallback.

The mod takes over their job. In routed mode, it replaces the memory prompt section and denies or redirects memory-dir writes itself. In fallback mode, it calls `next(e)` and steps aside.

## Deciding the mode

- **At `session.start`:** probe the vault with `$.mcp.call(VAULT, 'stats')` or a similar cheap call, under a short timeout. Store the mode in `$.state`.
- **Per call:** a routed `tool.call` hook whose vault call throws or returns `isError` falls through to `next(e)` (native) and flips the mode to fallback. This builds on how guards already work: a failed hook is skipped and the tool runs. So the `.catch` falls through with `next(e)` instead of denying.
- **The prompt section:** `prompt.section` answers are cached for the thread. Switching mode mid-thread only takes effect after `$.ui.invalidate("prompt.section")` plus a new thread or a compaction, unless the answer carries `breakPromptCache: true`. Use `breakPromptCache` only when the mode actually flips.
- **While in fallback:** re-probe on a timer (`$.clock`) or at each `turn.complete`. When the vault answers, migrate, then switch to routed.

## Migration

The source is whatever native memory holds that the vault doesn't:

- auto memory: `~/.claude/projects/<project>/memory/*.md` (`MEMORY.md` plus topic files)
- subagent memory: `~/.claude/agent-memory*/<name>/`, `.claude/agent-memory*/<name>/`
- the memory-store tools, if this surface has them: `memory_list`, then `memory_read` for each entry

Steps, all idempotent, so a crash halfway through is safe to rerun:

1. **Find the files.** List the native files. Skip any whose content hash is already in `$.store` under `migrated`.
2. **Write.** Write each one to the vault. Default destination: a `claude-memory/inbox/<project>/<file>` folder. Sorting into `kb/facts`, `kb/rules` and so on is a follow-up, done by a person or an agent following the vault's conventions. If the destination already exists with different content, append instead of overwriting.
3. **Record.** Put the content hash in `$.store.migrated` only after the vault write succeeds.
4. **Leave the original.** Don't delete it. Replace it with a one-line pointer to its vault path, the same way the manual migration of 2026-11-06 handled files (originals kept as `*.bak-premigrate`). `MEMORY.md` stays a valid index.

Migration runs:

- at `session.start` when the vault is up and native files have changed since the last run (mtimes or hashes)
- on the fallback-to-routed transition mid-session

## Open

- **Show the mode.** Should fallback be visible? A status line (`$.ui.status('memory: native (vault down)')`) or a toast on each flip would do it.
- **Read-your-own-writes.** During fallback the model reads native memory. Right after migration, does it need the vault copy, or is the pointer file enough?
- **Mindmeld.** It's read-only from Claude Code's side (search), so it has no migration path. During fallback, retrieval just goes without it.
