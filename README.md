# dev-flow

A Claude Code plugin for planning and finishing changes.

| Command | What it does | Who starts it |
|---|---|---|
| `/dev-flow:lets-do` | Plans a task with clean-code and modularity built in, plus a quality-suite gate | You only |
| `/dev-flow:team-do` | Splits work into parallel workstreams and runs them with an agent team | You only |
| `/dev-flow:quality-suite` | Runs the full pre-merge review chain on the branch, fixing as it goes | You, or Claude when you ask to get a change ready to merge |
| `/dev-flow:clean-code` | Readability pass: naming, function size, comments, error handling | You, or Claude when relevant |
| `/dev-flow:code-modularizer` | Splits large or mixed-purpose files into small, focused modules | You, or Claude when relevant |

The short forms (`/lets-do`, `/team-do`, ...) usually work too, unless another skill has the same name. `/simplify`, `/code-review` and `/security-review`, which quality-suite runs, are built into Claude Code.

A typical change: `/dev-flow:lets-do` to plan it, implement the plan, `/dev-flow:quality-suite` to polish it, then commit and merge. For work that splits into independent pieces, `/dev-flow:team-do` does all of that in parallel.

## Install

In Claude Code:

```
/plugin marketplace add danielemoraschi/claude-dev-flow
/plugin install dev-flow@dev-flow
```

To get a newer version later: `/plugin marketplace update dev-flow`, then start a new session. To try local changes, add a clone instead: `/plugin marketplace add /path/to/claude-dev-flow`.

## lets-do

Plans the task the way you'd want it built. Before designing anything, Claude reads the clean-code and code-modularizer skills, on every run, and says so with their line counts. The plan then has a fixed shape, so the decisions are visible and a missing one stands out:

- **design decisions**: each clean-code and modularity decision, the rule it applies and the step it lands in, so you can see where the skills shaped the plan;
- a **file map**: every new or changed file, the one concept it holds, its measured current size and its expected size, with any file that would grow too big split as part of the plan;
- in **every step**: *structure* (files touched, what gets extracted), *code* (the functions and types, each named for one job), *errors* (how failures are handled and where) and *tests* (named for the behavior they prove).

It ends with a quality-suite pass as the quality gate.

```
/dev-flow:lets-do add CSV export to the reports page
/dev-flow:lets-do implement docs/specs/webhooks.md
/dev-flow:lets-do fix the race in the session token refresh
/dev-flow:lets-do split src/billing/invoice.ts by concern, no quality-suite
/dev-flow:lets-do add rate limiting to the public API; ignore code-modularizer
```

- Anything in the task overrides the defaults: "ignore code-modularizer", "skip clean-code", "no quality-suite".
- The project's CLAUDE.md wins over the generic advice (file-size limits, layering, naming).
- With no task, it asks what to do.

## team-do

Runs a piece of work as an [agent team](https://code.claude.com/docs/en/agent-teams), with your session as the lead:

1. The lead finds the work (your task, a plan file, or the plan already in the conversation) and splits it into **workstreams**. Each workstream owns its files and can be built without waiting on another one's unfinished code. Work that others depend on goes in an earlier wave.
2. It writes down the contracts where workstreams meet, sizes the team, and shows you the breakdown before spawning anyone.
3. Each workstream gets a **planner** teammate (Opus, high effort). The planner follows lets-do for its workstream and gives the plan a second correctness pass.
4. A **reviewer** teammate (Opus, high effort) then attacks each plan independently: wrong assumptions about the code, missed edge cases, contract breaks, vague structure or names, over-engineering. It works through graded findings with the planner in at most two rounds, and the lead settles anything still disputed. The lead then reviews all the plans together for consistency across workstreams.
5. Each workstream then gets an **implementer** teammate (medium effort). The implementer builds the plan in its own worktree, runs quality-suite on it and commits on its own branch.
6. The lead checks each branch against its plan (file map, design decisions, contracts), merges the approved ones in dependency order and runs lint, typecheck and tests. A workstream that can't be settled stays on its branch as **needs-human**, with the decision you need to make; the rest still merge. Pushing and opening a PR are left to you.

```
/dev-flow:team-do implement the plan above
/dev-flow:team-do docs/plans/billing.md
/dev-flow:team-do docs/plans/billing.md, 4 teammates
/dev-flow:team-do one workstream per module in the plan
/dev-flow:team-do add audit logging: frontend / backend / tests as workstreams, 3 teammates
/dev-flow:team-do the plan above; Sonnet for implementation, just go
/dev-flow:team-do the plan above, no worktrees, leave the branches unmerged
/dev-flow:team-do small refactor across three packages, no plan review
```

The argument is free text: the work plus any options, in plain words.

- **Team size**: "N teammates" caps how many workstreams run at once (each runs a planner, then an implementer, plus a reviewer while its plan is reviewed). Without it, the lead runs every parallel workstream at once, up to 5, and runs the rest in waves. If only one workstream can run at a time, it says a team isn't worth it and points you to lets-do.
- **Workstreams**: name them ("one per module", "frontend / backend / tests") and the lead uses yours instead of its own split.
- **Models**: name one per role ("Sonnet for implementation", "Opus for everything").
- **Effort** comes from the agent definitions in `agents/` and can't be changed per run. It applies to in-process teammates only; split-pane teammates use the lead's effort.
- **"no plan review"** drops the reviewer; plans go straight to the lead after the planner's second pass.
- **"just go"** skips the confirmation. **"no worktrees"** keeps everyone in one checkout, on their own files. **"leave the branches unmerged"** still checks and approves each workstream, then lists the branches instead of merging them.

Requirements and tips:

- Agent teams are experimental: set `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (shell or settings.json `env`) and run Claude Code interactively. Without it, team-do stops and tells you.
- Workstreams branch from your current commit, so commit anything they depend on first. On the default branch, the lead creates a `team/...` branch to integrate on.
- Teammate permission prompts show up in the lead's session: pre-approve common commands (tests, git worktree, your build tool) before a long run.
- A team costs many times the tokens of one session; use it for work that genuinely splits.

## quality-suite

Runs the pre-merge chain on the current branch's changes, end to end, without stopping between steps. The steps, in order:

1. code-modularizer
2. `/simplify`
3. `/security-review`, only when the diff touches auth, untrusted input, shell, SQL, network or dependencies
4. `/code-review high --fix`
5. clean-code
6. lint, typecheck and tests

Each step runs again while it keeps finding material issues, up to 3 passes, with fresh subagents for repeat passes and large diffs. When it hands back, only commit and merge are left.

```
/dev-flow:quality-suite
/dev-flow:quality-suite only the last commit
/dev-flow:quality-suite against develop
/dev-flow:quality-suite what landed in a1b2c3d
```

Or just ask:

```
run the quality suite
polish this branch
get this ready to merge
```

- The scope defaults to everything on the branch since it left the base branch: commits, staged and unstaged changes, and new files.
- Running it on a commit that's already merged reviews just that change, and puts the fixes on a new short-lived branch.
- It asks you only for decisions that are genuinely yours, such as a change to public API or two designs with real trade-offs. Everything else it fixes.
- The report starts with `COMPLETE` or `INCOMPLETE`. It never commits, pushes or merges.

## clean-code and code-modularizer

The guidance lets-do and quality-suite build on. Both also work on their own, and Claude can pick them up when relevant.

```
/dev-flow:clean-code src/api/handlers/orders.ts
/dev-flow:clean-code review my uncommitted changes for readability
/dev-flow:code-modularizer src/components/Dashboard.tsx
/dev-flow:code-modularizer where should the new notifications code live?
```

```
this file is getting hard to manage, split it up
make this function readable
```

code-modularizer targets 100-200 lines per file and splits before about 300. Both defer to limits and conventions in your CLAUDE.md.

## The team-do agents

team-do spawns three agents that ship in `agents/`:

| Agent | Model, effort | Job |
|---|---|---|
| `dev-flow:workstream-planner` | Opus, high | Plans one workstream with lets-do, second correctness pass, answers the reviewer |
| `dev-flow:workstream-plan-reviewer` | Opus, high | Attacks one plan independently and reports a verdict |
| `dev-flow:workstream-implementer` | lead's model, medium | Builds the approved plan in its own worktree, runs quality-suite, commits on its branch |

They're written as teammates. Each expects a brief from a lead (goal, owned files, contracts, base commit, the plugin's skills directory, paths) and reports back to that lead. You can still use them directly as subagents if you give them that brief yourself:

```
use the dev-flow:workstream-planner agent to plan the export module: it owns src/export/,
must not touch src/reports/, base commit HEAD, skills directory <path>/skills,
write the plan to /tmp/export-plan.md
```

The reviewer is the most useful one on its own: point it at any plan, including one from lets-do, for an adversarial second opinion. Without a planner to talk to, it returns its graded findings to you.

```
use the dev-flow:workstream-plan-reviewer agent on /tmp/export-plan.md: goal is CSV export
from the reports page, owns src/export/, base commit HEAD, skills directory <path>/skills
```

For planning in your own session, lets-do is usually the better fit. The planner is lets-do run for one workstream of a larger split, on Opus at high effort, without talking to you, with a second correctness pass and a review round required. The implementer has no skill counterpart.

## License

MIT, see [LICENSE](LICENSE).
