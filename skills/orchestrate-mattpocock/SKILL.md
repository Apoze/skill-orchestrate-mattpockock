---
name: orchestrate-mattpocock
description: Relay Matt Pocock implementation tickets sequentially, each in a fresh top-level Codex session. Use when the user explicitly asks to orchestrate or continue tickets produced by to-tickets from local Markdown or GitHub, in ChatGPT Desktop on macOS/Windows or Codex CLI on Linux.
---

# Orchestrate Matt Pocock tickets

Run a **relay**: one ready ticket, one fresh top-level implementer, one trusted
terminal result. Keep the orchestrator on the control plane; inspect neither
code nor tests.

## 1. Build the queue

1. Read the repository-root `AGENTS.md` or `CLAUDE.md`, then
   `docs/agents/issue-tracker.md`. Treat them as the authority for the tracker
   kind and target.
2. Resolve tickets from the user's argument, the current `/to-tickets`
   context, or the single unambiguous ready set:
   - For local Markdown, use `.scratch/<feature>/issues/` only when the tracker
     document configures it.
   - For GitHub, resolve the exact `owner/repo` from an explicit ticket URL,
     repository guidance, or Git remotes. Prefer `remote.pushDefault`, the
     branch `pushRemote`, `origin`, then the sole GitHub remote. Ask when
     candidates remain ambiguous.
   - Pass `--repo <owner/repo>` to every `gh issue` or `gh api` read. Resolve
     child groups from `Parent` references or native sub-issues; exclude the
     parent specification from implementation tickets.
   - Build dependency edges from `Blocked by` references or native
     dependencies.
3. Record each ticket's identifier, title, canonical file path or issue URL,
   status, and blockers.
4. Count as complete only tickets explicitly closed/done, accepted by this
   relay, or declared complete by the user. Do not infer completion from code,
   diffs, tests, commits, or Git history.
5. Select one ready ticket at a time. Break ties by identifier or filename.

Show the detected count and ordered queue before launch. Continue only when
every selected reference resolves and every blocker names a known ticket. A
user-supplied starting ticket overrides earlier progress.

Completion criterion: the queue and dependency frontier are explicit and
unambiguous.

## 2. Prepare each implementer

Resolve these skills project-first, including `.agents/skills`, then from the
user's global installations:

- `implement`
- `ponytail`
- `code-structure`

Record each absolute `SKILL.md` path. Return
`REQUIRED_SKILLS_UNAVAILABLE: <names>` when any is missing. Global installation
is optional.

Record one stable `RELAY_SOURCE` for the whole run: use the current task URL or
CLI session ID when available; otherwise generate a UUID and retain it in the
orchestrator conversation. Use the same value in every child prompt.

Route the ticket from its text and acceptance criteria. Use
`route-codex-task` when available, otherwise `high`. Stop on `SPEC_REQUIRED` or
`SPLIT_REQUIRED`. Preserve the configured model; route only the reasoning
effort.

Completion criterion: the ticket, three required skill paths, and safe effort
are resolved before any child is created.

## 3. Select exactly one Codex surface

Choose by capability, not by the checkout's operating system:

1. When Codex App thread handlers are exposed in ChatGPT Desktop, read and follow
   [`references/desktop-app.md`](references/desktop-app.md). This branch has
   priority, including a Desktop remote project hosted on Linux.
2. Otherwise, only when the orchestrator itself runs in Codex CLI on Linux and
   `codex exec --help` succeeds, read and follow
   [`references/linux-cli.md`](references/linux-cli.md).
3. Otherwise stop with `EXECUTION_SURFACE_UNAVAILABLE` and the failed
   capability check.

On Desktop macOS/Windows, a broken App handler surface requires a fresh or
forked Desktop task; it does not switch to CLI. Use the App's native thread
tools there. On Linux CLI, use native `codex exec` sessions. Both branches use
the same prompt and result contract below.

Completion criterion: exactly one surface reference is loaded and its
preflight passes.

## 4. Launch prompt

Create one top-level child with the routed effort and no model override. Use
this exact structure, resolving every placeholder and preserving Codex's
explicit skill-link syntax:

```text
[$implement](<absolute implement SKILL.md path>) ticket <ticket identifier>

Implement only this ticket:
<ticket identifier> — <ticket title>
<absolute ticket path or canonical issue URL>

RELAY_SOURCE: <stable orchestrator task URL, CLI session id, or relay UUID>

Follow the ticket, its linked specification, and the normal $implement workflow.

Mandatory supporting skills:
[$ponytail](<absolute ponytail SKILL.md path>)
[$code-structure](<absolute code-structure SKILL.md path>)

Follow ponytail throughout implementation. Apply code-structure before deciding
whether this ticket should introduce or refactor shared operational logic. If a
required skill cannot be loaded, return BLOCKED.

Finish with exactly one terminal block:

IMPLEMENT_RESULT: PASS | BLOCKED | FAILED
TICKET: <ticket identifier>
COMMIT: <commit sha or NONE>
SUMMARY: <one concise line>

Use PASS only after the normal $implement workflow completed and committed the
ticket. Use BLOCKED for user input or an external prerequisite. Use FAILED for
failed implementation or validation.
```

Title Desktop children `Implement <identifier> — <title>`. Keep completed
children visible. Never create a replacement for an active or uncertain
ticket.

## 5. Relay terminal results

Track only these states in the orchestrator conversation:

- `READY`: no child launch has been attempted for the ticket.
- `CREATION_UNKNOWN`: launch may have created a child, but no identifier was
  returned.
- `ACTIVE`: one child identifier is recorded.
- `CHILD_UNKNOWN`: the recorded child temporarily cannot be observed.
- `TERMINAL`: one final result was read.

Treat `CREATION_UNKNOWN` and `CHILD_UNKNOWN` as transport states, never ticket
results. Follow the selected surface's recovery rules; creation is called at
most once per ticket.

Accept a ticket only when its latest final answer contains one unique terminal
block and all of these hold:

- `IMPLEMENT_RESULT` is exactly `PASS`;
- `TICKET` matches the launched ticket;
- `COMMIT` is present and not `NONE`.

On acceptance, record the ticket, child/session URL or ID, and commit; recompute
the frontier; then launch the next ticket through the surface's normal fresh
creation path. Reset all recovery state between tickets.

Stop on `BLOCKED`, `FAILED`, an explicit unresolved request to the user, an
invalid terminal block, or exhausted transport recovery. Report only the
observed result or exact request and the child/session identifier. Perform no
independent code read, test, review, repair, or tracker mutation.

When reinvoked in the same orchestrator conversation, resume its recorded
active child before creating anything. Finish only after every selected ticket
has an accepted `PASS`, then return the ordered ticket, child/session, and
commit list.
