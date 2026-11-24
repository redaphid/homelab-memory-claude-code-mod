# Mirror design: native memory runs, the vault gets a copy

Requirement, 2026-11-23, in two steps:

1. "We will want to 'fall back' to default behavior, and migrate on fallback."
2. Refined: "maybe we intercept the tool call and add the memory to my markdown vault, then let it continue default behavior."

So the mod is a **tee**. Claude Code's own memory works exactly as it does by default. Every memory write also goes to the vault, so the vault holds a full copy that every harness can read.

```
 model ── memory write ──▶ mod hook ──▶ next(e): native write runs (unchanged)
                               │
                               └──▶ on success: copy the written file to the vault
                                       │ vault down?
                                       └──▶ queue it; a catch-up scan copies it later
```

Why this beats routing memory to the vault instead of native memory:

- **Fallback is free.** Native memory always runs. A vault outage costs nothing but a delayed copy.
- **Migration becomes catch-up.** A catch-up scan copies whatever the vault is missing. There's no mode to switch and no prompt-cache invalidation.
- **Little to get wrong.** The mod never answers in place of a tool, so it never has to fake a tool's result schema.

## What it hooks

| Native write | Hook | What gets copied |
|---|---|---|
| Auto memory, through `Write` / `Edit` / `NotebookEdit` into `~/.claude/projects/<p>/memory/` | `tool.call` on those tools, path-matched | The whole file as it stands after the tool ran, read back with `$.fs` (an `Edit` input only carries a diff) |
| Subagent memory, the same tools into `agent-memory*/<name>/` | Same | Same |
| `memory_write` (surfaces that have it) | `tool.call` `{ tool: 'memory_write' }` | `e.content` |
| Anything else: `Bash` writes, deletes, files written while the mod was off | The catch-up scan | Whatever differs |

The rule for every hook:

1. Call `await next(e)` first.
2. Copy only if that result is not a `deny` and not `isError`. Then the vault never holds a write that didn't land natively.
3. Return the native result untouched.

```ts
on('tool.call', { tool: 'Write' }, async ($, e, next) => {
  const ran = await next(e)
  if (MEMORY_DIR.test(e.file_path) && ran.deny === undefined && !ran.isError)
    await mirror($, e.file_path).catch(() => queue($, e.file_path))
  return ran
})
```

`mirror` runs under a short timeout so a slow vault never stalls the tool call. On failure the path goes into a `$.store` queue for the catch-up scan.

## Catch-up scan (the "migration")

It runs at `session.start`, after a failed copy, and on a timer while the queue is not empty. It is idempotent:

1. List the native memory files: auto memory, agent memory, and `memory_list` where that tool exists.
2. Hash each one. Skip any whose hash matches the last successful copy recorded in `$.store`.
3. Write it to the vault, then record the hash.
4. A file that disappeared natively is **not** deleted from the vault. Mark it removed in its vault frontmatter instead.

The first run copies everything existing native memory holds, which makes it the one-time migration too.

## Where copies land

Default: `claude-memory/<project>/<file>` in the vault, one folder per Claude Code project. The vault is the copy and native is the source, so a later native write overwrites the vault copy. Sorting copies into `kb/facts`, `kb/rules` and so on is separate work, done by a person or an agent following the vault's conventions.

## Consequences

- **Native memory turns back on.** Today `autoMemoryEnabled: false`, `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` and a `PreToolUse` guard that denies memory-dir writes keep it off. All three go once the mod ships, or there's nothing to copy.
- **One-way sync.** Edits made in the vault don't flow back into native memory. Claude Code keeps reading its own copy.
- **Retrieval is separate.** The mirror doesn't make Claude Code read the vault or mindmeld. If that's wanted, add a line to the memory prompt section (`prompt.section`, append, don't replace) or inject hits once with `prompt.context`.

## Considered and set aside: routing to the vault, with fallback

The first version of this doc had the mod answer memory calls from the vault (`{ result }` in place of the tool), switch to native when the vault was down, and migrate native writes when it came back. That needs mode tracking, prompt-section swaps that fight the prompt cache, and results faked to each tool's schema. The mirror gets the same outcome (vault has everything, nothing lost during an outage) with none of that.
