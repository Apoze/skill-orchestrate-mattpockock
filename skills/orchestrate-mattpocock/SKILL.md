---
name: orchestrate-mattpocock
description: Orchestrate Matt Pocock implementation tickets sequentially in fresh Codex App threads.
---

# Orchestrate Matt Pocock tickets

Run the implementation frontier as a **relay**: one ticket, one fresh top-level
thread, one trusted terminal result.

## 1. Build the queue

1. Read the repository-root `AGENTS.md` or `CLAUDE.md`, then
   `docs/agents/issue-tracker.md`. Treat these files as the authority for the
   tracker kind and target.
2. Resolve the ticket set from the user's argument, the current `/to-tickets`
   context, or the single unambiguous ready set:
   - use `.scratch/<feature>/issues/` only when the tracker document explicitly
     configures local Markdown;
   - for GitHub, resolve the exact `owner/repo` from an explicit ticket URL or
     repository named in the user context, `AGENTS.md`, `CLAUDE.md`, or the
     tracker document. Otherwise inspect Git remotes, preferring
     `remote.pushDefault`, then a branch `pushRemote`, then `origin`, then the
     sole GitHub remote. Ask when multiple candidates remain;
   - pass `--repo <owner/repo>` to every `gh issue` or `gh api` read. Never rely
     on `gh`'s default repository or on `gh repo view` without `--repo`: a fork
     may track an upstream remote whose issue list is unrelated;
   - list the configured GitHub repository's ready issues and resolve child
     ticket groups from their `Parent` references or native sub-issue links.
     Do not include a referenced parent spec as an implementation ticket. If
     exactly one ready child group exists, use it; ask when groups are
     ambiguous. Use `Blocked by` references or native dependencies for edges.
3. Record each ticket's identifier, title, full file path or canonical issue
   URL, status,
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
Report an empty queue only after querying the explicitly resolved tracker
target and accounting for its ready child tickets.

## 2. Require the execution surface

Require the installed `$implement`, `$ponytail`, and `$code-structure` skills
and these Codex App tools, or their current equivalents:

- `codex_app__list_projects`
- `codex_app__list_threads`
- `codex_app__create_thread`
- `codex_app__wait_threads`
- `codex_app__read_thread`
- `codex_app__set_thread_title`

Discover skills from the project first, including `.agents/skills`, then from
the user's global installation. Record each resolved absolute `SKILL.md` path;
global installation is not required. Return `REQUIRED_SKILLS_UNAVAILABLE` with
the missing names when any required skill is absent. Codex App thread tools are
injected directly by Codex Desktop; they are not connector tools and
`tool_search` cannot load them. Check the callable tool surface, including
`ALL_TOOLS` when using `functions.exec`, and use the exposed App tools directly.
If any required App tool is absent, stop with `APP_THREADS_UNAVAILABLE` and
`ACTION: START_OR_FORK_FRESH_CODEX_THREAD`. A task can retain an older tool
surface across turns, and a skill cannot repair it. Do not retry in that task.
This relay has no plugin MCP, collaboration subagent, or `codex exec` branch.

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
ticket values and resolved skill paths. Keep the explicit skill-link syntax
produced by the Codex skill picker; do not replace it with `/implement` or a
natural-language request:

```text
[$implement](<absolute implement SKILL.md path>) ticket <ticket identifier>

Implement only this ticket:
<ticket identifier> — <ticket title>
<absolute ticket path or issue URL>

Follow the ticket, its linked spec, and the normal implement workflow.

Mandatory supporting skills:
[$ponytail](<absolute ponytail SKILL.md path>)
[$code-structure](<absolute code-structure SKILL.md path>)

Follow ponytail throughout the implementation. Apply code-structure before
deciding whether this ticket should introduce or refactor shared operational
logic. Apply it only where appropriate; do not create an abstraction when its
own rules say not to.
If any required skill cannot be loaded, return BLOCKED.

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
treat the outcome as uncertain and never retry creation. Reconcile for up to
three consecutive observations: list recent threads on the current host, then
read candidates in the current working directory whose initial prompt contains
the exact ticket reference and this relay's source thread. Repeat when no match
is visible because App thread creation may appear after the error response.
Resume the single match. If there is still no match after three observations,
wait three minutes without creating anything, then reconcile once more. Stop
only if that final observation has no match, or if more than one match appears.
A title or later App error must not discard an already recorded child
identifier.

## 4. Relay the terminal result

Wait with `codex_app__wait_threads` using a three-minute timeout and its cursor
for subsequent waits. While the child is active, inspect only the compact
thread/turn status returned by `wait_threads`; ignore commentary, emit no
progress summaries, and do not call `read_thread`. On timeout, wait again.
Never end the relay turn while the recorded child is active. When the child
completes or needs attention, read only its latest final answer with
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
