# Third-party patchers (ccmod, tweakcc)

Checked on 2026-11-24 by a research agent. Verdict: **don't use them for this mod.** The official mods API covers the whole [sync design](sync.md). A binary patcher only adds one thing, editing Anthropic's own prompt text, and it breaks on updates.

## ccmod

- What it is: a Python binary patcher by `idiolectai`, with one release, 1.0.0, on 2026-04-17 ([PyPI](https://pypi.org/project/ccmod/)). Its GitHub repo now returns 404.
- How it works: it finds fixed phrases in the JavaScript bundled inside the binary, rewrites the code between them, and re-applies itself through a SessionStart hook after updates.
- Versions: it targets Claude Code 2.1.101–2.1.111, so it very likely doesn't work on 2.1.331.
- What it does: turns off A/B prompt injections and extracts the prompt. We found no tool interception or tool injection.

## tweakcc

- What it is: by Piebald-AI, 2,619 stars, last push 2026-11-21, npm 4.4.0 ([repo](https://github.com/Piebald-AI/tweakcc)).
- How it works: on native installs it extracts the bundled JavaScript, patches it and repacks the binary.
- Versions: its general patches are verified only on 2.1.162. It says it fetches prompt data for new releases within minutes.
- What it does: rewrites system-prompt sections and hides built-in tools through "toolsets". Arbitrary code goes in through `adhoc-patch`, which makes no backup.
- Some of its options weaken safety checks: skipping the trust dialog, skipping permission checks for skills, and bypassing permissions under sudo.

## Compared with the official mods API

| | Patchers | Mods API |
|---|---|---|
| Edit or delete Anthropic's own prompt text | Yes | Replace or drop a whole section (`prompt.section`); not edit inside it |
| Hide built-in tools | tweakcc toolsets | `tool.call` deny; agent `disallowedTools` |
| Intercept a tool call and answer it | No | Yes |
| Call MCP servers from the hook | No | `$.mcp.call` |
| Survives Claude Code updates | No: re-patch every release | Yes: a supported API |
| Safety | Modifies the binary; some options weaken safety checks | Runs within the engine's hook budgets and permissions |
