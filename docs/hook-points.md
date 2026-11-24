# Where a mod can intercept memory

Checked on 2026-11-23 against Claude Code 2.1.331. The API facts come from the engine's own type file (`types/claude-code.d.ts`, written by the `plugin-authoring` skill) and the [mods docs](https://code.claude.com/docs/en/plugins/mods/events.md). The code below is an **untested sketch**.

## Short answer

Yes, a mod can hook a tool call by tool name, before the tool runs. It can also answer the call itself, so the native tool never runs and the model gets the mod's answer as that tool's result.

## Mod layout

This mod follows [draw-mod](https://github.com/redaphid/draw-mod):

```
.claude-plugin/plugin.json        { "name": ..., "version": ..., "types": "./types/index.d.ts" }
.claude-plugin/marketplace.json   so it installs with /plugin install <mod> --marketplace <owner>/<repo>
hooks/hooks.json                  { "modules": ["./register.ts"] }
hooks/register.ts                 export const register: Register = (on, options) => { ... }
types/index.d.ts                  the $.state contract, if the mod keeps state
```

Every hook has the shape `($, e, next)`:

- `$` is the engine interface.
- `e` is the event input.
- `next(e)` runs the hooks beneath this one and then the engine's own behaviour.

A hook that returns without calling `next` answers for itself.

## The events that matter

| Event | Fires | What the mod can do |
|---|---|---|
| `tool.call` with `{ tool: '<name>' }` | Before a tool runs, before the permission prompt | Return `{ deny: reason }`, rewrite the input with `next({ ...e, ... })`, or return `{ result }` in place of the tool |
| `tool.run` (2.1.326+) | Once the call is allowed | Also accepts `{ result }` |
| `prompt.section` with `{ name }` | Once per named system-prompt section, as the prompt is put together | Return `{ text }` to replace the section, or `{ text: null }` to drop it |
| `prompt.compose` | When the whole system prompt is rendered | Append, replace, reorder or drop sections |
| `prompt.context` | Once per conversation, for the context blocks of the first user message | Append or rewrite blocks, for example vault search hits |
| attachment injection | Per message the engine injects (a reminder, a mentioned file, `nested_memory`) | `{ text: null }` leaves it out |
| `agent.spawn` | When a subagent starts | Remove `memory:`, carry the vault contract |
| `session.start` | Session start and every reload | Register commands, set up state |

`$.mcp.call(server, tool, args)` calls a tool on any connected MCP server over the engine's own connection, with no permission prompt. That is how the mod reaches the vault and mindmeld from inside a hook.

### Notes from the type file

- **The result is checked.** A hook's `{ result }` is validated against the tool's output schema when the tool has one, put into the model's words by the tool's own mapper, and recorded in the transcript as the tool's result. For `memory_write`, it must match `{ outcome, path, store_kind?, version?, ... }`.
- **Guards fail open.** A hook that throws or overruns its budget is skipped, and the tool runs. Give a guard a `.catch(($, e, next) => next.called ? next(e) : { deny: '...' })`.
- **Prompt sections are cached.** After `$.ui.invalidate("prompt.section")`, a changed answer waits for a new thread or a compaction, unless it carries `breakPromptCache: true`.
- **Section ids.** The type file's own example is `on("prompt.section", { name: "memory" }, () => ({ text: null }))`. The public docs don't list section ids, so read them from `next(e)` in a `prompt.compose` hook before relying on `memory`.

## Sketch

```ts
import type { Register } from 'claude-code'

const VAULT = 'soul'                       // MCP server name as /mcp lists it
const MEMORY_DIR = /\/\.claude\/(projects\/[^/]+\/memory|agent-memory(-local)?)\//

const CONTRACT = `Memory lives in the markdown vault (MCP ${VAULT}), not in this harness.
Search it before relying on recall; write durable lessons back the same turn.`

export const register: Register = on => {
  // 1. Replace the native memory instructions with the vault contract.
  on('prompt.section', { name: 'memory' }, () => ({ text: CONTRACT }))

  // 2. Answer memory_write from the vault; the native store never sees it.
  on('tool.call', { tool: 'memory_write' }, async ($, e) => {
    const r = await $.mcp.call(VAULT, 'markdown-vault-mcp__write', {
      path: `claude-memory${e.path}`,
      content: e.content,
    })
    return { result: { outcome: r.isError ? 'failed' : 'ok', path: e.path } }
  }).catch(($, e, next) => (next.called ? next(e) : { deny: 'vault write failed' }))

  // 3. Writes into the old memory directories are refused, with the way forward.
  on('tool.call', { tool: 'Write' }, ($, e, next) =>
    MEMORY_DIR.test(e.file_path)
      ? { deny: `Harness memory is off. Write to the vault (MCP ${VAULT}) instead.` }
      : next(e),
  )
}
```

`memory_read` and `memory_list` map the same way, to vault `read` and `list_documents` or `search`. The exact vault tool names depend on how the MCP gateway prefixes them. Check them with `/mcp`.

## Classic hooks vs the mod API

| | `settings.json` hooks | Mod API |
|---|---|---|
| Match by tool name | Yes, a regex matcher on `PreToolUse` | Yes, `{ tool }` on `tool.call` |
| Deny | Yes, `permissionDecision: deny` | Yes, `{ deny }` |
| Rewrite the input | Yes, `updatedInput` | Yes, `next({ ...e, ... })` |
| Answer in place of the tool | **No** | **Yes**, `{ result }` without `next` |
| Replace the output after the tool ran | `PostToolUse` with `updatedToolOutput` / `updatedMCPToolOutput` | `await next(e)`, then return something else |
| Change the system prompt | No | `prompt.section`, `prompt.compose` |
| Call an MCP tool | Only by shelling out to a client | `$.mcp.call` |
| Ships as an installable plugin | No, it's hand-edited per machine | Yes, through a marketplace |

Classic hooks can only block a memory write or change its input, and the model has to retry against the vault itself. The mod can answer the call from the vault directly.
