# /delegate: a frontier model manages, Grok workers build

A Cursor **skill** plus two **worker subagents**. Type `/delegate` before a
multi-step task (an app, a game, a refactor, a code review). The model you're
chatting with then works as the project manager:

1. it plans the work and splits it into small, clearly briefed,
   self-contained pieces;
2. it runs Grok 4.7 subagents in parallel: `grok-worker` for edits and
   `grok-reviewer` (read-only) for reviews, audits, and codebase maps;
3. it reviews every result (accepts it, sends it back, or takes it over);
4. it assembles and verifies the final product.

The manager keeps the work where a frontier model earns its price: planning,
interface design, judgement, catching subtle bugs, and integration. The bulk
work goes to Grok workers that each start with a clean context. That covers
reading lots of files, boilerplate, well-specified modules, tests, mechanical
edits, and per-file review.

> Adapted from a community "/delegate" skill that used Claude Opus as the
> manager and Claude Haiku as the workers. Its author reported roughly 40%
> lower total cost across three builds, with the biggest savings on
> reading-heavy work like code reviews. That figure is the original author's
> claim. It hasn't been measured for this Grok version.

## What's inside

```text
skills/delegate/SKILL.md   # the manager playbook, invoked as /delegate
agents/grok-worker.md      # editing worker, model: grok-4.7-high
agents/grok-reviewer.md    # read-only worker (readonly: true), model: grok-4.7-high
README.md
LICENSE
```

## Recommended setup

| Role | Model | Why |
| --- | --- | --- |
| Manager | The strongest reasoning model you have, e.g. Claude Opus 5.5, GPT-5.6 Sol, or Claude Fable 5.1 | Planning and review quality decide the outcome. |
| Workers | Grok 4.7, **standard speed**, `high` effort | Cheap, fast, large context. Fast mode doubles the price. |

- **Manager:** pick it in the chat's model picker before typing `/delegate`.
  A weak manager writes vague briefs and accepts bad output, which wipes out
  the savings.
- **Workers:** the agent files pin `grok-4.7-high` (high effort, standard
  speed). Some Cursor versions ignore the variant in local chats and give
  subagents Grok 4.7 at whatever effort and speed are selected for it in your
  model picker (for example `grok-4.7-high-fast`). To be safe, also set
  Grok 4.7 in the picker to **high effort, Fast off**, then switch back to
  your manager model. Fast is the default on Premium and the legacy
  Pro/Pro+/Ultra plans, and it bills workers at 2x.

## Install (Cursor)

**For all your projects (user level):**

```bash
# macOS / Linux
mkdir -p ~/.cursor/skills ~/.cursor/agents
cp -r skills/delegate ~/.cursor/skills/
cp agents/*.md ~/.cursor/agents/
```

```powershell
# Windows (PowerShell)
New-Item -ItemType Directory -Force "$HOME\.cursor\skills","$HOME\.cursor\agents" | Out-Null
Copy-Item -Recurse -Force skills\delegate "$HOME\.cursor\skills\"
Copy-Item -Force agents\*.md "$HOME\.cursor\agents\"
```

**For one repo (project level, shared with your team via git):**

```bash
mkdir -p .cursor/skills .cursor/agents
cp -r /path/to/cursor-skills-delegate/skills/delegate .cursor/skills/
cp /path/to/cursor-skills-delegate/agents/*.md .cursor/agents/
```

Restart Cursor (or open a new chat). Check that `delegate` appears under
**Customize → Skills**, and that `grok-worker` and `grok-reviewer` appear as
subagents. Project subagents in `.cursor/agents/` take precedence over user
ones with the same name.

## Usage

```text
/delegate build a small Pomodoro web app: timer, task list, localStorage, tests
/delegate review the changes on this branch against main, file by file
/delegate migrate all API handlers in src/api/ from callbacks to async/await
/delegate map this codebase and propose a plan to add multi-tenant support
```

The skill has `disable-model-invocation: true`, so it runs only when you
type `/delegate`. It never kicks in on its own.

**Choosing models per run.** After showing the plan, the manager asks you
which model the workers and reviewers should use, with the same
multiple-choice prompt as Plan mode. The options come from the models your
Cursor offers subagents (Grok and Composer only), with the agent-file pin
`grok-4.7-high` recommended first. To skip the question, name the models in
your message:

```text
/delegate build the settings page, workers on composer, reviewers on grok 4.7
```

## Which Grok model?

Both agent files set `model: grok-4.7-high`. As of October 2026 Grok 4.7 is
the newest Grok model in Cursor (Grok 4.6 and 4.5 are still available).

- Grok 4.7 in Cursor: 256k standard context, 500k long context. Additional
  usage costs $2 / $0.50 / $6 per 1M input / cache-read / output tokens, or
  $4 / $1 / $12 on Fast. **Above 256k input tokens a request bills at 2x**
  (3x on Fast). Grok models draw from your plan's included pool of Cursor
  models on every paid plan.
- Effort levels are `low | medium | high | xhigh` (default `high`). The skill
  uses `high` by default, `medium` for mechanical pieces, and `xhigh` only to
  retry a hard piece that failed.
- When a newer Grok ships, change the `model:` line in both agent files. If
  your Cursor rejects the variant pin (the task card shows a non-Grok model),
  fall back to a plain `grok-4.7`.
- Model IDs (from `agent models` in the Cursor CLI, October 2026): Grok 4.7 is
  `grok-4.7-<effort>` and `grok-4.7-<effort>-fast`, with effort `low`,
  `medium`, `high` or `xhigh`. Older versions are `cursor-grok-4.6-*` and
  `cursor-grok-4.5-*`. A plain `grok-4.7` is also accepted.
- Grok doesn't take bracket parameters. The CLI rejects
  `grok-4.7[fast=false]`, `grok-4.7[effort=high,fast=false]` and
  `grok-4.7[reasoning_effort=high,fast=false]`. To pin effort and speed, use
  the hyphenated ID instead, e.g. `grok-4.7-high` (high effort, standard
  speed).

## Tips: keep workers small

- **Stay far below the 256k long-context threshold.** The skill keeps each
  worker's context under about 50k tokens. Smaller briefs are cheaper and
  faster, and the worker drifts less.
- One concern per brief, with an explicit list of the files it owns.
  Workers running at the same time must own **disjoint files**. They share
  one checkout and will overwrite each other.
- Paste the exact interfaces and snippets into the brief instead of telling a
  worker to "read the module". Spell out edge cases: workers follow briefs
  literally.
- Reading-heavy jobs (code review, audits, mapping a codebase) benefit most.
  Run one `grok-reviewer` per file or diff chunk, then let the manager dedupe
  and verify the findings.
- Don't delegate tiny tasks. Writing briefs and starting workers has overhead.
  If it's 2–3 edits, let the main agent just do it.
- Keep security- and correctness-critical code (auth, crypto, concurrency,
  money, migrations) with the manager. Workers can write its tests.
- Workers never commit, push, deploy, send anything, or launch their own
  subagents. The manager (and you) stay in control.

## Known issues / troubleshooting

- **Workers don't run on Grok.** Cursor falls back to another model when your
  plan or team admin doesn't allow the pinned one. On legacy request-based
  plans without Max Mode, subagents always run Composer. The skill tells the
  manager to check the task card on the first wave and, if the model isn't
  Grok, to pass a Grok slug from the Task tool's list explicitly. If no Grok
  model is offered at all, the manager stops and asks you instead of quietly
  running workers on its own, more expensive model.
- **Workers run on Fast and cost 2x.** Turn Fast off for Grok 4.7 in the
  model picker (see Recommended setup). A sign that Fast is on: the Task
  tool's model list offers `grok-4.7-high-fast` but not `grok-4.7-high`.
  When it sees this, the manager mentions it in the model question and
  labels each `-fast` option `(Fast, 2x cost)`.
- **Reviewers say their shell is blocked.** That's expected: `readonly: true`
  blocks the shell as well as edits. Reviewers list what they couldn't
  confirm under UNSURE, with a command, and the manager runs it.
- **Two workers edited the same file.** Tighten the FILES YOU OWN lists, or
  run those pieces in separate waves.

## License

MIT, see [LICENSE](LICENSE).
