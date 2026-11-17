---
name: capture-lesson
description: Turn a session finding into a durable artifact by choosing the right destination, showing the exact diff, and waiting for approval. Use when a session revealed something worth keeping — a tool or server behaviour, an environment gotcha, a correction the user made, a preference, or a procedure worth repeating — or when asked to "remember this", "capture this lesson", "update memory", "record this correction", or "make this stick". Do not use for a one-off task detail with no reuse value.
---

# Capture a lesson

The learning loop is a human gate, not a background job. Nothing here writes a file until the
user approves the exact text.

That is not caution for its own sake. Hook firing is per event here, so a capture that waits for
`postToolUseFailure` never happens (measured 2026-11-15; per-event table in
`docs/reference/environment.md`). This skill is the same loop driven on purpose, at the moment the
finding is still in context.

## 1. Decide whether it is durable

Keep it only if both are true:

- **It survives the session.** A file name, a line number, or today's target is not durable.
- **It will change a future action.** If the answer would be the same next time without this
  note, do not record it.

Do not capture: the task result, a transitory error that was retried successfully, or a restatement
of something already written elsewhere. Point at the existing note instead — a second copy becomes
a second, silently-stale truth.

## 2. Choose the destination by half-life

| Destination | Half-life | Put here | Never put here |
|---|---|---|---|
| **User memory** `…\memories\*.md` | a year | who the user is, how they want to be worked with, universal rules | project facts, dated logs, anything over ~40 lines |
| **Repo memory** `…\memories\repo\<PROJECT>\` | a month | verified project facts: system, client, transport, object behaviour, decisions | secrets, credentials, source code |
| **Skill** `~/.copilot/skills/<name>/` | until the method changes | a procedure worth repeating the same way twice | one-off answers, project facts |
| **Agent file** `~/.copilot/agents/*-by-pongg.agent.md` | until the tools change | routing, safety gates, evidence rules | history, anecdotes, project data |
| **Nothing** | — | ephemeral detail | — |

The order matters. User memory is loaded into **every** request, so it is the most expensive place
and holds only what is true of all work. Project facts belong in repo memory, which is read on
demand. When user memory is close to its budget, prefer repo memory even for a general-feeling
fact.

Ownership is by provenance, not by folder. Editable: a file ending `-by-pongg`, a skill you
created, and this kit's five skills (`capture-lesson`, `long-task-control`, `mcp-surface-audit`,
`new-program`, `rap-lifecycle`) — their source of truth is the kit repo, so edit the clone and
reinstall. Read-only: other installer-owned skills (SAP AI Dev Toolkit) and any other file under
`~/.copilot` — say so and stop.

## 3. Require evidence, then write the exact text

State, in one line each:

- **Fact** — what you now know.
- **Evidence** — the command output, file read, or tool result from this session that proves it.
  No evidence, no capture. A hypothesis may be recorded only if labelled a hypothesis.
- **Scope** — where it applies: this machine, this workspace, this project, this SAP client.
- **Expiry** — what would make it wrong (a server update, a new client, a reinstall).

Then show the diff — the exact lines added, changed, or removed, with the file path — and wait for
approval. Do not paraphrase the diff; show it.

## 4. Keep the destination honest

- **Date version-specific claims.** `verified 2026-11-13` tells a future reader how much to trust it.
- **Correct, do not stack.** If the new fact contradicts an existing line, replace that line and
  say what was wrong. Two contradictory lines are worse than one outdated line.
- **Prefer a pointer.** If the rule already lives in an agent or a skill, the memory entry states
  where it lives and stops.
- **Measure the budget before adding.** User memory holds roughly 200 lines:

```powershell
$m="$env:APPDATA\Code\User\globalStorage\github.copilot-chat\memory-tool\memories"; $t=0; Get-ChildItem $m -File | ForEach-Object { $t += ((Get-Content $_.FullName) -split "`r`n").Count }; "used=$t budget=200 headroom=$((200-$t))"
```

If the answer is under about 20 lines of headroom, consolidate or archive before adding.

## 5. Close the loop

Report what changed and where, in a table:

| File | Change | Why |
| --- | --- | --- |

If the finding came from a failure, also say whether the same failure is now preventable and how.
A lesson that changes nothing is a note, not a lesson — say that plainly rather than implying
progress.
