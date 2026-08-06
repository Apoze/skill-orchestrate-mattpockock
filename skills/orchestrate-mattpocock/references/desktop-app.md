# ChatGPT Desktop transport

Use this branch for the Codex task surface in ChatGPT Desktop on macOS or
Windows, including remote projects. The positive target is one native, visible,
top-level App task per ticket.

## Preflight

Discover these exact handlers through the current task's tool registry:

- `codex_app__list_projects`
- `codex_app__list_threads`
- `codex_app__create_thread`
- `codex_app__wait_threads`
- `codex_app__read_thread`
- `codex_app__set_thread_title`

In one tool-orchestration call, type-check all six and make real read-only calls
to `list_projects`, `list_threads`, `read_thread` on the current task or one
reachable listed Codex task, and `wait_threads` with `timeoutMs: 0`. Decode JSON
strings before validating their shape. A returned string containing
`No handler registered for tool:` is an error even when the promise resolves.
Rejection only because the probe tried to wait on its own calling task proves
that `wait_threads` is registered.

`create_thread` and `set_thread_title` are type-checked only because trial calls
would mutate App state. Mark the preflight complete only after every read-only
handler responds and both mutating handlers exist.

On failure, return the exact error plus:

```text
APP_THREADS_UNAVAILABLE
ACTION: START_OR_FORK_FRESH_CODEX_THREAD
```

Run this preflight once per relay task. Resolve the current App project by
canonical working directory and host, then keep that binding while the relay
continues. Ask only when multiple matches remain.

Completion criterion: `APP_THREADS_READY` is established from actual handler
calls, not discovery metadata alone.

## Create one child

Call `create_thread` once with the resolved project, the shared prompt, the
routed effort as `thinking`, and no `model`. Use `environment.type: local` so
the child shares the same checkout. Immediately record the returned task and
host identifiers, then set its title.

If creation throws, returns any App error (including `No handler registered`),
or omits an identifier, set `CREATION_UNKNOWN`. An error response does not
prove that creation failed. Call no second `create_thread` for that ticket.

After three minutes, make one compact reconciliation:

1. List recent tasks on the resolved host.
2. Match canonical working directory, exact ticket reference, and the
   `RELAY_SOURCE` value in the initial prompt.
3. Adopt one exact match as `ACTIVE`; stop on multiple matches.
4. With no match, wait another three minutes and retry, for at most three
   observations. Then stop with `CREATION_UNRESOLVED`.

A returned child identifier remains authoritative even if title-setting or a
later App call fails.

Completion criterion: one recorded child, or an explicit unresolved creation
result after bounded reconciliation; never a duplicate.

## Wait cheaply

Use `wait_threads` only for the recorded child. A logical polling interval is
three minutes. On Windows, implement it as a 120-second wait followed—only on
timeout—by a 60-second wait using the returned cursor. Read nothing between
those waits. On timeout, repeat the same pattern. Keep commentary and progress
out of the relay response.

When the child completes, call `read_thread` once for its latest final answer.
Treat `needs-attention` only as a wake signal: read the latest turn and stop
only for a visible unresolved approval or user-input request addressed to the
user. With no such request and no terminal block, retain the child and cursor
and return to the three-minute wait.

After an `ACTIVE` child's wait, read, or host lookup fails, set
`CHILD_UNKNOWN`. Preserve its identifier and cursor. Let three minutes elapse
without inspecting it, then make one recovery attempt:

1. Read the exact identifier without `hostId`.
2. If needed, list recent tasks and rebind the exact identifier, or the single
   candidate matching working directory, `RELAY_SOURCE`, and ticket reference.
3. If still unknown, wait three minutes before repeating. Call no creation
   tool during recovery.

Use timing tools in quiet chunks when an App wait handler is temporarily
missing; perform no child read between recovery intervals. Stop only after
three failed recovery observations with `CHILD_UNRESOLVED`.

After a terminal result, discard the cursor, recovered host binding, and retry
history. The next ticket always starts with the normal `create_thread` path.

## Desktop guardrails

- Use native App task handlers for creation and observation.
- Use shell commands only for repository/tracker reads or quiet timing.
- Keep one child active at a time and leave completed tasks unarchived.
- A fresh/forked Desktop task is the recovery for an unavailable handler
  surface.

Do not start a raw `codex app-server` client, a collaboration subagent, a
`codex exec` child, or Computer Use on this branch.
