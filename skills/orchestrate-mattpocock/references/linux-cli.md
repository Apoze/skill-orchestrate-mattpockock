# Codex CLI Linux transport

Use this branch only when the orchestrator itself runs in Codex CLI on Linux
and native App task handlers are unavailable. The positive target is one
persistent `codex exec` session per ticket.

## Preflight

Confirm all of the following before the first launch:

- `uname -s` reports `Linux`.
- `codex exec --help` exits successfully.
- `codex login status` confirms usable authentication.
- The canonical working directory is a Git repository.
- The selected noninteractive permission policy can edit the worktree, run the
  ticket's validation, access required network resources, and create the
  required Git commit.

Use the least-permissive existing policy that meets those requirements and do
not broaden beyond the orchestrator's current authority. Default
`workspace-write` protects `.git` on some systems; when no already-approved
noninteractive policy can commit, stop before launch with
`CLI_COMMIT_PERMISSION_REQUIRED` rather than silently granting full access.

The CLI currently documents reasoning efforts through `xhigh`. If routing
returns an effort rejected by `codex exec`, stop with
`CLI_REASONING_EFFORT_UNSUPPORTED`; do not silently lower it.

Completion criterion: `CLI_EXEC_READY` with working authentication, Git, route,
and permissions.

## Create one session

Create a private temporary directory with `mktemp -d`. Place the shared prompt,
JSONL events, stderr, and final result there; keep generated artifacts outside
the repository. Write the prompt without interpolating ticket content into a
shell command.

Start one persistent session (omit `--ephemeral`) and omit `--model`:

```bash
codex exec --json \
  -C <canonical-working-directory> \
  --sandbox <approved-noninteractive-sandbox> \
  -c 'approval_policy="never"' \
  -c 'model_reasoning_effort="<routed-effort>"' \
  --output-last-message <temporary-result-file> \
  - < <temporary-prompt-file> \
  > <temporary-events-file> \
  2> <temporary-stderr-file>
```

Use the environment's long-running process facility rather than blocking the
terminal. Read the first `thread.started` JSONL event once and record its
`thread_id` as the child identifier. Do not stream or summarize later JSONL
events.

If the process launch errors before a `thread.started` event, inspect the event
and stderr files once. With no session ID, stop with `CLI_CREATION_FAILED`. With
one session ID, preserve it as `ACTIVE`; starting another session would create
a duplicate.

Completion criterion: exactly one recorded persistent session ID.

## Wait cheaply

Poll only the operating-system process status at three-minute intervals. Do
not read JSONL progress, the worktree, code, tests, or Git history while it is
running. When the process exits, read the final-result file once.

If the process exits without a valid final result but has a recorded session
ID, set `CHILD_UNKNOWN`. Inspect stderr and the terminal JSONL event once. When
the failure is transport-only and resumable, run exactly one recovery for the
same child:

```bash
codex exec resume --json \
  -c 'approval_policy="never"' \
  -c 'sandbox_mode="<approved-noninteractive-sandbox>"' \
  -c 'model_reasoning_effort="<routed-effort>"' \
  -o <temporary-result-file> \
  <recorded-session-id> \
  'Continue the same ticket to the required terminal block.'
```

Read only the new final result after completion. Stop with
`CLI_CHILD_UNRESOLVED` when this single recovery does not produce a valid
terminal result.

After a terminal result, remove only that exact temporary directory. Keep the
Codex session persisted. For the next ticket, call a fresh `codex exec`; use
neither `resume --last` nor the previous session ID.

## Linux guardrails

- Use native `codex exec` and its persisted sessions.
- Use one session per ticket and one recovery at most for that same session.
- Keep prompts on stdin, machine events in JSONL, and the final response in the
  output-last-message file.
- Keep the orchestrator blind to implementation progress and repository
  changes.

Do not start a raw App Server client, a collaboration subagent, Desktop task
tool, or Computer Use on this branch.
