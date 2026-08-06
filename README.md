# Orchestrate Matt Pocock

Runs tickets produced by Matt Pocock's `/to-tickets` sequentially. Each ticket
gets a fresh top-level Codex session running `$implement`; the relay continues
only after the implementer reports a committed `PASS`.

## Requirements

- ChatGPT Desktop on macOS/Windows with native Codex top-level task tools, or
  Codex CLI on Linux with working `codex exec`
- Matt Pocock's `implement` skill
- `ponytail` and `code-structure`
- `route-codex-task` is optional; without it, implementers use `high`

This skill does not depend on `codex-project-orchestrator`.
It uses no plugin MCP or custom App Server client.
Linux CLI sessions remain available through `codex resume`; they do not appear
as Desktop sidebar tasks.

## Install

From the project that already contains the Matt Pocock skills:

```bash
npx skills@latest add Apoze/skill-orchestrate-mattpockock \
  --skill orchestrate-mattpocock --agent codex -y
```

Add `-g` for a user-level installation.

For a manual project installation, copy
`skills/orchestrate-mattpocock` beside `.agents/skills/implement`.

Invoke it explicitly:

```text
$orchestrate-mattpocock
```

To resume after manually completed tickets:

```text
$orchestrate-mattpocock start at ticket 03
```
