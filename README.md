# /delegate: your agent manages, Grok workers build

A Cursor **skill** plus a **worker subagent**. Type `/delegate` before a
multi-step task (an app, a game, a refactor, a code review). Your main model
then works as the project manager:

1. it plans the work and splits it into small, clearly briefed,
   self-contained pieces;
2. it runs `grok-worker` subagents, pinned to the newest Grok model, in
   parallel;
3. it reviews every result (accepts, sends back, or redoes);
4. it assembles and verifies the final product.

The main model keeps the expensive work (planning, judgement, integration).
The bulk work (reading lots of files, boilerplate, mechanical edits, per-file
review) goes to Grok workers that each start with a clean context.

> Adapted from a community "/delegate" skill that used Claude Opus as the
> manager and Claude Haiku as the workers. Its author reported roughly 40%
> lower total cost across three builds, with the biggest savings on
> reading-heavy work like code reviews. That figure is the original author's
> claim. It hasn't been measured for this Grok version.

## What's inside

```text
skills/delegate/SKILL.md   # the orchestrator (manager playbook), invoked as /delegate
agents/grok-worker.md      # the worker subagent, model: grok-4.7
README.md
LICENSE
```

## Install (Cursor)

**For all your projects (user level):**

```bash
# macOS / Linux
mkdir -p ~/.cursor/skills ~/.cursor/agents
cp -r skills/delegate ~/.cursor/skills/
cp agents/grok-worker.md ~/.cursor/agents/
```

```powershell
# Windows (PowerShell)
New-Item -ItemType Directory -Force "$HOME\.cursor\skills","$HOME\.cursor\agents" | Out-Null
Copy-Item -Recurse skills\delegate "$HOME\.cursor\skills\"
Copy-Item agents\grok-worker.md "$HOME\.cursor\agents\"
```

**For one repo (project level, shared with your team via git):**

```bash
mkdir -p .cursor/skills .cursor/agents
cp -r /path/to/delegate-skill/skills/delegate .cursor/skills/
cp /path/to/delegate-skill/agents/grok-worker.md .cursor/agents/
```

Restart Cursor (or open a new chat). Check that `delegate` appears under
**Customize → Skills**. Project subagents in `.cursor/agents/` take
precedence over user ones with the same name.

## Usage

```text
/delegate build a small Pomodoro web app: timer, task list, localStorage, tests
/delegate review the changes on this branch against main, file by file
/delegate migrate all API handlers in src/api/ from callbacks to async/await
```

The skill has `disable-model-invocation: true`, so it runs only when you
type `/delegate`. It never kicks in on its own. Pick whichever model you want
as the manager in the model picker. The workers use the model pinned in
`grok-worker.md`.

## Which Grok model?

`grok-worker.md` sets `model: grok-4.7`. As of October 2026 that's the newest
Grok model in Cursor and on the xAI API.

- Cursor: 256k standard context, 500k long context. Additional usage costs
  $2 / $0.50 / $6 per 1M input / cached / output tokens. **Above 256k input
  tokens a request bills at 2x** (3x on Fast). Fast costs 2x the standard
  rate. Grok 4.7 is one of Cursor's own models, so it draws from your plan's
  included pool on every paid plan.
- When a newer Grok ships, change the `model:` line, e.g. to the new id shown
  in Cursor's models list.
- Optional parameters use square brackets, e.g.
  `grok-4.7[reasoning_effort=high]`. Grok 4.7 uses `reasoning_effort`, not
  `effort`, and supports `low | medium | high | xhigh`. A plain `grok-4.7` is
  the safest pin. See the known issues below.

## Tips: keep workers small

- **Stay far below the 256k long-context threshold.** The skill targets under
  roughly 50k input tokens per worker run. Smaller briefs are cheaper, faster,
  and drift less.
- One concern per brief, with an explicit list of the files it owns.
  Workers running at the same time must own **disjoint files**, or they will
  overwrite each other.
- Paste the exact interfaces and snippets into the brief instead of telling a
  worker to "read the module".
- Reading-heavy jobs (code review, audits, mapping a codebase) benefit most:
  one read-only worker per file or diff chunk, and the manager dedupes and
  checks the findings.
- Don't delegate tiny tasks. Writing briefs and starting workers has overhead.
  If it's 2–3 edits, let the main agent just do it.
- Workers never commit, push, deploy, or send anything. The manager (and you)
  stay in control.

## Known issues / troubleshooting

- **The pinned model isn't used.** Cursor falls back when your plan or team
  admin doesn't allow the model, and on legacy request-based plans without
  Max Mode. In October 2026 there were also forum reports of local subagents
  pinned to Grok 4.7 falling back to another model, or rejecting
  effort/fast variants. Cursor says the inherit/fallback part is fixed. A
  remaining limitation is that the local Task tool only offers Grok 4.7 at the
  reasoning level currently selected in your model picker (e.g.
  `grok-4.7-high`). If workers don't run on Grok 4.7, set Grok 4.7 to the
  effort you want in the picker. The skill also tells the manager to fall back
  to the newest Grok variant the Task tool lists, and to report which model
  actually ran (the subagent task card shows it).
- **Two workers edited the same file.** Tighten the FILES YOU OWN lists, or ask
  for isolated worktrees ("…each in its own environment").

## License

MIT, see [LICENSE](LICENSE).
