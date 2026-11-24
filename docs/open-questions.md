# Open questions

As of 2026-11-23.

## Unknown

- **Which surfaces expose `memory_list`, `memory_read` and `memory_write`?** They are in the 2.1.331 engine but not in a local CLI session's tool list, and Anthropic's docs and CHANGELOG don't name them. To find out: hook `tool.call` on them in a loaded mod, log any call, and check the tool list on claude.ai and on Claude Code on the web.
- **What is the memory section's id?** The type file's example uses `memory`, and the docs don't list ids. To find out: log the ids from `next(e)` in a `prompt.compose` hook, with auto memory on and with it off.
- **Can team memory be switched off locally?** We don't know yet.

## Answered

- **ccmod / tweakcc** (2026-11-24): third-party binary patchers. Not used. See [patchers.md](patchers.md).

## Decisions

- **Sync is the current direction** (2026-11-23): writes mirror to the vault, reads prefer the vault and backfill. See [sync.md](sync.md).
- **Where do redirected writes land in the vault?** One `claude-memory/` folder, or routed by type (`kb/facts`, `kb/rules`, `kb/research`) the way the vault's conventions expect?
- **Retrieval.** Should `prompt.context` inject vault or mindmeld hits at conversation start? Forced retrieval on every prompt was rejected earlier. Once per conversation is untested.
