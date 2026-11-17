---
name: long-task-control
description: Run, checkpoint and stop a task that outlives one request: the stop condition, the handoff when stopping early, and why evidence expires. Use when work spans multiple objects or requests, near a request or context limit, when resuming a previous session, or asked "what is left". Not for work that fits in one turn.
---

# Long-task control

A task that spans more than one turn fails in three specific ways, and each has a cheap defence.

## 1. State the stop condition before the work starts

At the start of a slice that will run long, say: **"I stop when X is activated and verified, or when
the next object needs a decision I cannot make."**

An undefined stop condition means the work ends whenever the request limit arrives — which is neither
the right time nor a reason anyone chose.

## 2. Escalate planning only for real reasons

Use lightweight planning by default. Escalate to an explicit plan when the work:

- spans multiple related objects or layers
- introduces a new RAP service or data model
- changes an interface or persistence contract
- touches enhancements
- has unclear business rules
- carries meaningful regression risk

Present a short plan and the decisions needed — not a long interview. For work that exceeds one
session, split it into independently verifiable slices with dependencies and clear completion
criteria. Do not create external tickets, branches, or delegate work unless the user asks and the
environment supports it.

### Delegating part of it

Delegation buys context isolation and parallelism, nothing else. It is worth it when the question
is independent (neither side waits), wide (many objects, a package, a long dump), and the brief is
cheaper than the work.

Do it yourself when the steps are serial — many subagents do not make an ordered job faster, only
longer — when the task is smaller than its brief, when you would re-verify the answer anyway (that
is a lead, not a result), and always when it is a write. When two investigations genuinely are
independent, running them one after the other wastes wall-clock time for no gain in care.

## 3. Stopping before completion: write a handoff, not a status report

A status report says what happened. A handoff lets the next session continue. It needs exactly:

1. the goal, in one sentence
2. what is done
3. what is not done
4. the next exact step
5. which facts must be re-verified, because their evidence is no longer in context

Point 5 is the one that gets left out and the one that causes the most damage next session.

A todo entry proves a step happened; it never proves the step succeeded. So a handoff that says "all
steps complete" and lists no evidence has told the next session nothing it can act on.
