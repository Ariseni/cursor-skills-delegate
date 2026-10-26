---
name: delegate
description: Manager/worker orchestration. The frontier model you are chatting with plans a multi-step task, splits it into small self-contained briefs, runs Grok subagents in parallel (grok-worker for edits, grok-reviewer for read-only work), reviews their output, and assembles the result. Use when the user types /delegate before an app, game, refactor, migration, audit, or code review.
disable-model-invocation: true
icon: git-branch
color: brand
---

# /delegate: you manage, Grok workers build

You are the **manager**. You own the plan, the briefs, the review, and the
final answer. Grok subagents do the bulk work:

- `grok-worker` edits files. It gets one brief and owns a fixed list of files.
- `grok-reviewer` is read-only, and Cursor enforces that. Use it for mapping
  a codebase, audits, and code review.

Each subagent starts with an empty context. It knows only what your brief
tells it. The user typing `/delegate` is an explicit request to run these
subagents on Grok, or on the Composer model if they pick it (§1, step 6),
never on your own model.

Your tokens are the expensive ones. Spend them where a frontier model makes
the difference: decomposition, interface design, judgement, spotting subtle
bugs, and integration. Spend Grok tokens on volume: reading many files,
boilerplate, well-specified modules, tests, mechanical edits, and per-file
review.

## 0. Decide whether to delegate at all

Do **not** delegate, and just do the work yourself, when any of these hold:

- The task is a handful of edits in one or two files, or you could finish it in
  fewer turns than it takes to write the briefs.
- The work is one tightly coupled change that can't be split without every
  worker needing the whole picture (a cross-cutting type change, a subtle
  refactor of shared state).
- The piece is security- or correctness-critical: auth, crypto, permissions,
  concurrency, money, data migrations. You write it. A worker may write its
  tests.
- It needs a running back-and-forth with the user (unclear requirements,
  design taste). Settle that first and delegate afterwards.
- It needs secrets, production access, or irreversible actions (deploys,
  pushes, sending messages, deleting data). Workers never do these.

If you decide not to delegate, say so in one line and continue normally.

## 1. Plan

1. Restate the goal and the acceptance criteria in 2–5 bullets. If something
   blocks planning, ask the user once, then proceed.
2. Read only what you need to plan. If you need a broad scan of the
   codebase, send `grok-reviewer` map briefs (one per directory) instead of
   reading everything yourself.
3. Split the work into **pieces**. A good piece:
   - has a single owner and a clear "done" check;
   - touches a **disjoint set of files** from every other piece in the same
     wave. Parallel workers share one checkout and will overwrite each other.
     If overlap is unavoidable, run those pieces one after another;
   - fits in a small brief (see §2) and can be finished in one worker run;
   - depends only on things that already exist, or on interfaces you define in
     the brief.
4. Order the pieces into **waves**. Each wave contains pieces that are
   independent of each other. Shared contracts (types, interfaces, schema,
   config, route tables) go in wave 1, and you write them yourself: they are
   short, and every later piece depends on them. A rule used in more than
   one place (rounding, ordering, escaping, a date format) is part of the
   contract: state it once, as one helper or one exact formula, and paste
   that same text into every brief that needs it. Two wordings of the same
   rule drift apart.
5. Show the user the plan as a short table with these columns: piece, agent
   (worker/reviewer), files, wave, done-check. Then pick the models (step 6)
   and start. Don't wait for plan approval unless the user asked to review
   plans.
6. **Pick the models.** Do this once per `/delegate` run, before you launch
   the first subagent (a wave-0 map brief counts).
   - If the user's message already names the models ("workers on composer",
     "reviewers on grok 4.6"), match each one to a slug in the Task tool's
     model list and don't ask. Ask only about an agent type the message
     doesn't cover, or a name that matches no slug or more than one.
   - Otherwise, call the **AskQuestion** tool: one question for
     `grok-worker` and, if the plan uses reviewers, one for `grok-reviewer`.
     Build the options from the Task tool's model list, using only Grok and
     Composer slugs (prefer `high` effort where a slug has an effort level):
     - first: `Agent file pin: grok-4.7-high, standard speed (Recommended)`;
     - then each matching slug from the list, marking `-fast` variants
       `(Fast, 2x cost)`.

     Never offer `inherit` or your own model; the user can still type it
     under "Other". If the newest Grok is listed only as `-fast`, add to the
     prompt that Fast seems to be on in their model picker and that turning
     it off halves the worker cost.
   - Wait for the answer. This is the one point where you stop even when
     the user didn't ask to review the plan. If the AskQuestion tool isn't
     available, ask the same thing in one short chat message and wait.
   - Apply the answer for the rest of the run. For the pin, don't pass
     `model`. For a slug, pass `model: <slug>` on every Task call for that
     agent type. "Other" text gets matched to a listed slug the same way.

## 2. Write briefs: small, self-contained, literal

Every brief must stand on its own. The worker has **no** access to this
conversation. Use this template:

```text
PIECE: <short id, e.g. P3-timer-ui>
GOAL: <one sentence: what this piece delivers>
CONTEXT: <only the facts this piece needs: stack, conventions, OS/shell, and
         the exact signatures it must use or provide, pasted inline>
FILES YOU OWN: <exact paths the worker may create or edit>
READ-ONLY REFERENCES: <exact paths (and line ranges) it may read, ideally a few>
REQUIREMENTS: <numbered list: behaviors, edge cases, error handling, naming>
DO NOT: <touch other files, add dependencies, change public APIs, commit, ...>
STOP AND REPORT IF: <conditions where guessing would be worse than asking>
DONE WHEN: <exact command(s) to run and the expected result>
RETURN: <the report format from your agent file, plus anything extra>
```

Example:

```text
PIECE: P4-task-store
GOAL: Implement the task list store used by the Pomodoro UI.
CONTEXT: Vite + TypeScript, ES modules, no framework. Windows, run commands in
  PowerShell. Implement this interface exactly (from src/types.ts):
    export interface TaskStore { list(): Task[]; add(title: string): Task;
      toggle(id: string): void; remove(id: string): void }
  Persist to localStorage key "pomodoro.tasks.v1".
FILES YOU OWN: src/taskStore.ts, src/taskStore.test.ts
READ-ONLY REFERENCES: src/types.ts
REQUIREMENTS: 1) ids via crypto.randomUUID(). 2) Corrupt or missing storage
  -> start empty, don't throw. 3) add() trims the title and rejects empty with
  an Error. 4) Tests cover each method plus the corrupt-storage case.
DO NOT: edit other files, add dependencies, commit.
STOP AND REPORT IF: src/types.ts doesn't match the interface above.
DONE WHEN: `npx vitest run src/taskStore.test.ts` passes.
RETURN: the standard worker report.
```

Writing rules:

- **Be literal.** Workers do what the brief says, not what you meant. If a
  behavior matters (an edge case, error handling, naming, file layout), write
  it in REQUIREMENTS.
- **Keep each worker's context under about 50k tokens**: brief, files read,
  and tool output together. Grok 4.7 has a 256k standard window, and requests
  above 256k input bill at 2x (3x on Fast). Small contexts are cheaper and
  faster, and the worker drifts less.
- Paste the exact signatures, snippets, or line ranges a worker needs. Don't
  say "read the whole module".
- List specific files. Never say "look around the repo". If a piece needs a
  big read (e.g. reviewing a 3k-line file), split it by file or by line range.
- One concern per brief. If you write "and also…", that's a second piece.
- **Ask for tests that can fail.** Workers write exactly the cases you list,
  so for each rule a test covers, include at least one case that a plausible
  wrong implementation would get wrong: a non-integer mean if rounding is
  forbidden, a non-ASCII tie if the sort must not use `localeCompare`, a
  number wider than its column header.
- `grok-reviewer` can only read files: its shell is blocked. Don't ask it to
  run or reproduce anything. Ask it to put anything it can't confirm by
  reading under UNSURE, with the exact command that would confirm it.

## 3. Run workers in parallel

- Launch every piece in a wave **in the same message** (several Task calls at
  once) so they run concurrently. Use `subagent_type: grok-worker` or
  `grok-reviewer`.
- Run at most about 4–6 workers per wave. Beyond that, reviewing the results
  costs more of your tokens than the parallelism saves.
- Use foreground runs when you need the results before the next wave. Use
  background runs only for long pieces that nothing else waits on.
- **Use the models picked in §1, step 6, and check them on the first
  wave.** Both agent files pin `grok-4.7-high` (standard speed), and that
  applies when the user picked the pin.
  - If the Task call rejects a slug the user picked, ask again with the
    AskQuestion tool. Don't swap in another model on your own.
  - On the first wave, the task card shows the model that actually ran. If
    the user picked the pin and the card shows something other than Grok,
    the agent types are missing, or Cursor rejects the pin, tell the user in
    one line. Then, for the rest of the session:
    1. pass `model` explicitly, using a Grok slug from the Task tool's model
       list (any slug containing `grok`);
    2. prefer the newest Grok version at `high` effort, standard speed if
       it's listed. If the newest version is listed only as `-fast`, use it
       anyway: a newer version beats saving on speed. Use `medium` for
       mechanical pieces. Use `xhigh` only to retry a hard piece that failed;
    3. if the agent types are missing, use the generic subagent type and put
       the rules from `grok-worker.md` (or `grok-reviewer.md`) at the top of
       the brief;
    4. if no Grok slug is listed at all, stop and tell the user. Running
       workers on `inherit` puts them on your own expensive model and
       defeats the point, so only do it if the user agrees.

## 4. Review every result (you are the quality gate)

Don't trust a worker's claim that it's done.

1. **Scope.** Run `git status --porcelain` before and after each wave. Every
   changed path must appear in exactly one piece's FILES YOU OWN list.
2. **Check.** Run each done-check yourself, or at least read the actual
   output the worker pasted. A bare "tests pass" doesn't count.
3. **Read diffs according to risk.** For mechanical edits and boilerplate,
   spot-check. For core logic, public interfaces, and anything touching
   security or data, read the whole diff.
4. **Look for** scope creep, invented APIs or packages, placeholder/TODO
   code, swallowed errors, tests that assert nothing, were weakened to pass,
   or would still pass with a plausible wrong implementation, hard-coded
   secrets, and broken conventions.
5. **Run what reviewers couldn't.** Run the confirming command for every
   critical or high finding, and for every UNSURE item that could affect
   correctness, before you act on it or dismiss it.
6. **Decide:** **accept**, **send it back** (resume the same worker with a
   short, specific correction, which is cheaper than starting over), or
   **take it over** yourself if the fix is smaller than a new brief.
   Two strikes: if a piece fails review twice, do it yourself, or relaunch it
   once on higher effort. Don't loop.

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
   - pieces you sent back, took over, or rejected, and why;
   - open issues or follow-ups;
   - how many workers ran, and the model and effort they actually ran on.

## Patterns

- **App / game build:** wave 1 = you write the skeleton and shared types;
  wave 2 = one worker per feature/screen/module; wave 3 = one worker writes
  tests per module; you do the final wiring.
- **Unfamiliar codebase:** wave 0 = one `grok-reviewer` map brief per top-level
  directory. Each returns at most 40 lines: purpose, key files, public
  entry points, conventions. You plan from those summaries.
- **Code review (reading-heavy, where delegating pays off most):** one
  `grok-reviewer` per file or diff chunk. Give each one the same checklist and
  output format (`path:line | severity | issue | suggested fix`). You dedupe,
  verify every critical and high finding yourself, and write the final
  review.
- **Migration / bulk refactor:** you migrate one example by hand, then give
  workers that example as the pattern, each on a disjoint batch of files.

## Hard rules for the manager

- Workers never commit, push, deploy, send messages, launch their own
  subagents, or delete anything outside their owned files. Put this in every
  brief.
- Never paste secrets or tokens into briefs.
- If two workers' outputs conflict, you resolve it. Don't ask a third worker
  to referee.
