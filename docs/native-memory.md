# Claude Code's native memory systems

Checked on 2026-11-23 against Claude Code 2.1.331 (native binary install). Sources:

- the mod API type file that the `plugin-authoring` skill writes (`types/claude-code.d.ts`, about 29,000 lines)
- `strings` on the binary, for the system-prompt text
- the official docs: [memory](https://code.claude.com/docs/en/memory), [sub-agents](https://code.claude.com/docs/en/sub-agents), [hooks](https://code.claude.com/docs/en/hooks), [mods events](https://code.claude.com/docs/en/plugins/mods/events.md)

Claude Code has five memory surfaces. The mod has to cover each one: turn it off, redirect it, or keep it on purpose.

| # | Surface | How it works | How to turn it off | Plan |
|---|---|---|---|---|
| 1 | Auto memory | Files under `~/.claude/projects/<project>/memory/`, written with the ordinary Write and Edit tools. A system-prompt section tells the model about it. | `autoMemoryEnabled: false` in settings, or `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` | Left on as the fallback; while the vault is up, the memory section is replaced with the vault contract |
| 2 | Team memory | A shared directory synced at session start with the other users of the project | Not checked | Off or redirected |
| 3 | Memory-store tools | Tools named `memory_list`, `memory_read` and `memory_write` | Unknown | Redirected to the vault, or hidden |
| 4 | Subagent memory | A subagent's `memory: user \| project \| local` frontmatter | Turned off along with auto memory | Off; the subagent gets the vault contract instead |
| 5 | CLAUDE.md files | Loaded as context. The engine counts them as "memory files". | — | Kept: this is how the pointer to the vault loads |

## 1. Auto memory

- **Writing.** The model writes memory files itself with its standard file tools. The docs list four note types: `user`, `feedback`, `project` and `reference`. Claude Code stamps a `modified` time on a memory file that has YAML frontmatter.
- **Loading.** The first 200 lines or 25 KB of `MEMORY.md` load at session start. Topic files are read on demand.
- **Prompt.** The binary has several variants of the system-prompt section, among them:
  - `You have a persistent, file-based memory system at <dir>.`
  - a variant with a private directory and a shared team directory
  - a read-only variant that says "If the user asks you to remember something, explain that memory is read-only in this session."
  - a one-fact-per-file variant with frontmatter
- **Settings.**
  - `autoMemoryEnabled` (any settings scope; `/memory` writes it to user settings). When `false`, "Claude doesn't read from or write to the auto memory directory".
  - `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1`, which wins over the setting for a session
  - `autoMemoryDirectory`, which redirects the directory but doesn't remove the feature
  - Self-hosted environment sessions default to off.

Turning auto memory off removes both the directory and the prompt section. That leaves nothing to fall back to, so the mod keeps auto memory **on** and replaces the prompt section itself while the vault is reachable. See [fallback-and-migration.md](fallback-and-migration.md).

## 2. Team memory

The binary has prompt text for "a persistent, file-based team memory directory ... synced at the start of every session and shared with the other users who work in this project". The changelog mentions team memory stores through `CLAUDE_MEMORY_STORES` in remote sessions. The docs we read say nothing about a local switch for it.

## 3. Memory-store tools (`memory_list`, `memory_read`, `memory_write`)

The engine's type file defines these built-in tools:

```ts
memory_list:  { store?: string; path_prefix?: string; cursor?: string }
memory_read:  { store: string; path: string }
memory_write: { store: string; path: string; content: string; if_version: string }
```

- Results carry `outcome` (`ok`, `not_found`, `conflict`, `missing`, `refused` or `failed`), `store_kind` (`personal` or `project`), and a 12-character `version` token for optimistic concurrency.
- A document is capped at 100 KB, and a write replaces the whole document.
- The binary also has telemetry event names `memory_write_opt_in_requested` and `memory_writes_setting_changed`, which suggests a user-facing setting.

**Not documented anywhere we could find**, not even in the CHANGELOG. The tools are absent from a local CLI session's tool list. We don't know which surfaces get them (claude.ai, Claude Code on the web and Managed Agents are all candidates). Managed Agents' documented memory stores are a mounted directory (`/mnt/memory/<slug>/`) accessed with the ordinary file tools, not these tools ([docs](https://platform.claude.com/docs/en/managed-agents/memory)).

## 4. Subagent memory

`memory: user | project | local` in an agent definition gives that subagent a directory:

- `user`: `~/.claude/agent-memory/<name>/`
- `project`: `.claude/agent-memory/<name>/`
- `local`: `.claude/agent-memory-local/<name>/`

The subagent's prompt gets read/write instructions plus the first 200 lines or 25 KB of its `MEMORY.md`. Turning auto memory off disables the field.

## 5. CLAUDE.md files

CLAUDE.md, rules files and imports arrive as context in a user message, not in the system prompt. The engine's types call them `ContextMemoryFile`, and nested ones arrive as `nested_memory` attachments. The mod keeps these, because a user CLAUDE.md is the simplest way to load the pointer to the vault in every harness.
