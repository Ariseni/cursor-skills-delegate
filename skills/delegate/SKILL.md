---
name: delegate
description: Manager/worker orchestration. The main agent plans a multi-step task, splits it into small self-contained briefs, runs grok-worker subagents (newest Grok model) in parallel, reviews their output, and assembles the result. Use when the user types /delegate before an app, game, refactor, migration, or code review.
disable-model-invocation: true
icon: git-branch
color: brand
---

# /delegate: you manage, Grok workers build

You are the **manager**. You own the plan, the briefs, the review, and the final
answer. The `grok-worker` subagents do the work. They run on the newest Grok
model, and each one starts with an empty context. They only know what your
brief tells them.

The point is to keep expensive reasoning (planning, judgement, integration) in
your context. Bulk work (reading lots of files, writing boilerplate, mechanical
edits, per-file review) goes to workers.

## 0. Decide whether to delegate at all

Do **not** delegate, and just do the work yourself, when any of these hold:

- The task is a handful of edits in one or two files, or you could finish it in
  fewer turns than it takes to write the briefs.
- The work is one tightly coupled change that can't be split without every
  worker needing the whole picture (a subtle concurrency fix, a cross-cutting
  type change).
- It needs a running back-and-forth with the user (unclear requirements,
  design taste). Settle that first and delegate afterwards.
- It needs secrets, production access, or irreversible actions (deploys,
  pushes, sending messages, deleting data). Workers never do these.

If you decide not to delegate, say so in one line and continue normally.

## 1. Plan

1. Restate the goal and the acceptance criteria in 2–5 bullets. If something
   blocks planning, ask the user once, then proceed.
2. Read only what you need to plan. If you need a broad scan of the
   codebase, give it to a worker as a read-only "map" brief instead of reading
   everything yourself.
3. Split the work into **pieces**. A good piece:
   - has a single owner and a clear "done" check;
   - touches a **disjoint set of files** from every other piece running at the
     same time (otherwise workers overwrite each other; if overlap is
     unavoidable, run those pieces one after another, or ask for isolated
     worktrees);
   - fits in a small brief (see §2) and can be finished in one worker run;
   - depends only on things that already exist, or on interfaces you define in
     the brief.
4. Order the pieces into **waves**. Each wave contains pieces that are
   independent of each other. Shared foundations (types, interfaces, schema,
   config) come first, in wave 1. Usually you write these yourself because
   they are short and decide everything else.
5. Show the user the plan as a short table: piece, files, wave, done-check.
   Then start. Don't wait for approval unless the user asked to review plans.

## 2. Write briefs: small, self-contained, explicit

Every brief must stand on its own. The worker has **no** access to this
conversation. Use this template:

```text
GOAL: <one sentence: what this piece delivers>
CONTEXT: <only the facts this piece needs: stack, conventions, the interfaces/
         signatures it must use or provide, pasted inline when short>
FILES YOU OWN: <exact paths the worker may create or edit>
READ-ONLY REFERENCES: <exact paths it may read, ideally a few>
DO NOT: <touch other files, add dependencies, run git commit/push, change public APIs, ...>
DONE WHEN: <concrete check: command to run, test to pass, behavior to show>
RETURN: <what to report: files changed, summary, check output, open questions>
```

Keep briefs small. It saves cost, and it stops a worker from drifting off the
task:

- **Target well under 50k input tokens per worker run.** That covers the brief
  plus the files it reads plus its own tool output. Grok 4.7 in Cursor has a
  256k standard window. Requests with more than 256k input tokens bill at 2x
  (3x on Fast), up to 500k. Never let a piece get anywhere near that.
- Paste the exact signatures, snippets, or line ranges a worker needs. Don't
  say "read the whole module".
- List specific files. Never say "look around the repo". If a piece needs a
  big read (e.g. reviewing a 3k-line file), split it by file or by line range.
- One concern per brief. If you write "and also…", that's a second piece.
- Tell the worker what to do when it gets stuck: stop and report, don't
  guess at scope.

## 3. Run workers in parallel

- Launch every piece in a wave **in the same message** (several Task calls at
  once) so they run concurrently. Use the `grok-worker` subagent type.
- Use foreground runs when you need the results before the next wave. Use
  background runs only for long, independent pieces.
- Model: the `grok-worker` agent is pinned to the newest Grok model. If that
  agent type isn't available, or Cursor rejects or falls back from its pinned
  model, launch a generic subagent with the brief prefixed by the worker rules
  from `grok-worker.md`. Then pick a model in this order:
  1. the newest Grok model the Task tool lists (e.g. `grok-4.7` or a variant
     like `grok-4.7-high`);
  2. otherwise the newest Grok variant that is listed;
  3. otherwise `inherit`.
  Tell the user which model actually ran (the task card shows it).
- Run at most about 4–6 workers per wave. More than that, and reviewing the
  results costs more than the parallelism saves.

## 4. Review every result (you are the quality gate)

Don't trust a worker's claim that it's done. For each result:

1. Check that it touched only the files it owned (`git diff --stat`, or read
   the files).
2. Run its done-check yourself, or check the output it pasted.
3. Skim the diff for scope creep, invented APIs, placeholder/TODO code,
   swallowed errors, secrets, and broken conventions.
4. Decide: **accept**, **send it back** (resume the same worker with a short
   correction, which is cheaper than starting over), or **redo it yourself**
   if the fix is smaller than a new brief.

Before you start the next wave, update any later briefs that depend on what
this wave actually produced.

## 5. Assemble and finish

1. Integrate the pieces: wiring, imports, small glue edits. You do this
   yourself, not a worker.
2. Run the project-level checks (build, lint, tests) once at the end.
   Delegate fixing failures only if they are many and independent.
3. Report to the user:
   - what was built or changed (by file or area);
   - the checks you ran and their results;
   - pieces you redid or rejected, and why;
   - open issues or follow-ups;
   - roughly how many workers ran, and on which model.

## Patterns

- **App / game build:** wave 1 = you write the skeleton and shared types;
  wave 2 = one worker per feature/screen/module; wave 3 = one worker writes
  tests per module; you do the final wiring.
- **Code review (reading-heavy, where delegating pays off most):** one
  read-only worker per file or diff chunk, each with the same review checklist
  and output format (`path:line | severity | issue | suggested fix`). You
  dedupe, check the high-severity findings yourself, and write the final
  review.
- **Migration / bulk refactor:** you migrate one example by hand, then give
  workers that example as the pattern, each on a disjoint batch of files.

## Hard rules for the manager

- Workers never commit, push, deploy, send messages, or delete anything outside
  their owned files. Put this in every brief.
- Never paste secrets or tokens into briefs.
- If two workers' outputs conflict, you resolve it. Don't ask a third worker
  to referee.
