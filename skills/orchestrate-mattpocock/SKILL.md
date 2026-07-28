---
name: orchestrate-mattpocock
description: Orchestrate Matt Pocock implementation tickets sequentially in fresh Codex App threads.
---

# Orchestrate Matt Pocock tickets

Run the implementation frontier as a **relay**: one ticket, one fresh top-level
thread, one trusted terminal result.

## 1. Build the queue

1. Read `docs/agents/issue-tracker.md`.
2. Resolve the ticket set from the user's argument, the current `/to-tickets`
   context, or the single unambiguous ready set:
   - local tracker: read the ticket files under
     `.scratch/<feature>/issues/`;
   - configured remote tracker: use its documented read-only listing commands.
3. Record each ticket's identifier, title, full file path or issue URL, status,
   and blocking edges.
4. Treat only tickets explicitly closed/done, tickets completed by this relay,
   and tickets the user explicitly says are complete as complete. Inspect no
   code, diff, tests, or Git history to infer completion.
5. Select ready tickets whose blockers are complete. Run one at a time; break
   ties by ticket identifier or filename.

If several ticket sets are plausible, ask the user to select one. If the user
supplies a starting ticket, accept their stated progress and begin there.

Show the detected count and ordered queue. Continue only when every selected
ticket has a resolvable reference and every blocking edge names a known ticket.

## 2. Require the execution surface

Require the installed `$implement` skill and these Codex App tools, or their
current equivalents:

- `codex_app__list_projects`
- `codex_app__list_threads`
- `codex_app__create_thread`
- `codex_app__wait_threads`
- `codex_app__read_thread`
- `codex_app__set_thread_title`

Return `IMPLEMENT_UNAVAILABLE` when `$implement` is absent. Return
`APP_THREADS_UNAVAILABLE` when the App thread surface is absent. Use that
surface directly: this relay has no plugin MCP, collaboration subagent, or
`codex exec` execution branch.

Use `codex_app__list_projects` to resolve the current project. Match its
canonical working directory and host; ask the user only if more than one match
remains. Every child uses this project with `environment.type: local`, so
sequential tickets share the same checkout.

Resolve the project immediately before each launch. Do not persist a
`projectId` in repository files: App project identifiers may become stale.

## 3. Route and launch one ticket

Route the ticket from its text and acceptance criteria:

- when `route-codex-task` is available, apply it to the DEV ticket and use its
  effort;
- otherwise use `high`;
- stop on `SPEC_REQUIRED` or `SPLIT_REQUIRED`.

Pass the effort as `thinking` to `codex_app__create_thread`. Preserve the
configured model by omitting `model`.

Create one top-level thread with this initial prompt, substituting the exact
ticket values:

```text
Use $implement to implement only this ticket:
<ticket identifier> — <ticket title>
<absolute ticket path or issue URL>

Follow the ticket, its linked spec, and the normal $implement workflow.
Finish with exactly one terminal block:

IMPLEMENT_RESULT: PASS | BLOCKED | FAILED
TICKET: <ticket identifier>
COMMIT: <commit sha or NONE>
SUMMARY: <one concise line>

Use PASS only after the normal $implement workflow completed and committed the
ticket. Use BLOCKED when user input or an external prerequisite is required.
Use FAILED when implementation or validation failed.
```

Title the thread `Implement <identifier> — <title>`. Record its thread and host
identifiers in the relay conversation before waiting. Never create a second
thread for a ticket already recorded as active or complete.

If creation times out, returns an App error, or does not return an identifier,
reconcile once before reporting failure: list recent threads on the current
host, then read candidates in the current working directory whose initial
prompt contains the exact ticket reference and this relay's source thread.
Resume the single match. If there is no match or more than one, stop and report
the ambiguity; never retry creation blindly. A title or later App error must
not discard an already recorded child identifier.

## 4. Relay the terminal result

Wait with `codex_app__wait_threads`; use its cursor for subsequent waits.
When the child completes, read its latest final answer with
`codex_app__read_thread`.

Treat `No Codex thread found` during wait or read as a transient host-binding
error, not as a ticket result. Retry `read_thread` once without `hostId`; if it
still fails, list recent threads and rebind the exact thread identifier, or the
single candidate matching the working directory, source thread, and ticket
reference. Retry this recovery at most three consecutive times. Stop only when
all attempts fail, and never launch the next ticket while the result is
unknown.

Accept a ticket only when the terminal block is unique and all of these hold:

- `IMPLEMENT_RESULT` is exactly `PASS`;
- `TICKET` matches the launched ticket;
- `COMMIT` is neither missing nor `NONE`.

On acceptance, record the commit in the relay conversation, recompute the
frontier from the recorded results, and launch the next ticket.

On `BLOCKED`, `FAILED`, a needs-attention state, an invalid block, or an
unrecovered App error, stop the relay. Report the ticket, terminal result, and
`codex://threads/<thread-id>`. Perform no independent verification or repair.

When the user invokes the relay again in the same conversation, resume the
recorded active child before creating anything. Completed child threads remain
visible and unarchived.

Finish when every selected ticket has an accepted `PASS`. Return the ordered
ticket, thread, and commit list.
