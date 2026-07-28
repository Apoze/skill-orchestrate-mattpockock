# Orchestrate Matt Pocock

Runs tickets produced by Matt Pocock's `/to-tickets` sequentially. Each ticket
gets a fresh top-level Codex App thread running `$implement`; the orchestrator
continues only after the implementer reports a committed `PASS`.

## Requirements

- Codex App with top-level thread tools
- Matt Pocock's `implement` skill
- `route-codex-task` is optional; without it, implementers use `high`

This skill does not depend on `codex-project-orchestrator`.

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
