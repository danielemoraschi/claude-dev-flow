# dev-flow

A Claude Code plugin for planning and finishing changes.

| Command | What it does | Who starts it |
|---|---|---|
| `/dev-flow:lets-do` | Plans a task with clean-code and modularity built in, plus a quality-suite gate | You only |
| `/dev-flow:team-do` | Splits work into parallel workstreams and runs them with an agent team | You only |
| `/dev-flow:quality-suite` | Runs the full pre-merge review chain on the branch, fixing as it goes | You, or Claude when you ask to get a change ready to merge |
| `/dev-flow:quality-suite-lite` | Same chain, one pass each and no independent reviewer: for small changes | You only |
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
- **not building**: what the plan deliberately leaves out, one line each, so you can veto a cut;
- a **file map**: every new or changed file, the one concept it holds, its measured current size and its expected size, with any file that would grow too big split as part of the plan;
- in **every step**: *structure* (files touched, what gets extracted), *code* (the functions and types, each named for one job), *reuses* (what it uses instead of writing new code, or "nothing to reuse" and what was checked), *errors* (how failures are handled and where) and *tests* (named for the behavior they prove).

Claude then checks the plan itself and has the `plan-reviewer` agent attack it (Opus, high effort), folding the fixes into the steps and adding a short review summary. A second round runs when a blocking finding was fixed or disputed, so a dispute is settled by a reviewer, not by the plan's author. If the agent isn't loaded yet (right after a plugin update, before a restart), a general-purpose subagent reads its file and acts as it. The review is skipped for trivial plans or with "no plan review". The plan ends with a quality-suite pass as the quality gate: `/dev-flow:quality-suite-lite` when the plan is small and contained, the full suite otherwise or when the change is risky, with the reason given; you approve it with the plan.

```
/dev-flow:lets-do add CSV export to the reports page
/dev-flow:lets-do implement docs/specs/webhooks.md
/dev-flow:lets-do fix the race in the session token refresh
/dev-flow:lets-do split src/billing/invoice.ts by concern, no quality-suite
/dev-flow:lets-do add rate limiting to the public API; ignore code-modularizer
/dev-flow:lets-do rename the config loader, no plan review
/dev-flow:lets-do add CSV export, question the scope
```

- Anything in the task overrides the defaults: "ignore code-modularizer", "skip clean-code", "no quality-suite", "no plan review", "question the scope" (it pushes back on parts of the task the need doesn't justify before planning).
- It plans the smallest complete change: for each piece it takes the first option that fully works (not building it at all, something already in the codebase, the standard library, an installed dependency, one line, and only then new code), while still reaching every caller, test and fixture the change breaks. It never cuts validation, error handling, security or accessibility.
- The project's CLAUDE.md wins over the generic advice (file-size limits, layering, naming).
- With no task, it asks what to do.

## team-do

Runs a piece of work as an [agent team](https://code.claude.com/docs/en/agent-teams), with your session as the lead:

1. The lead finds the work (your task, a plan file, or the plan already in the conversation) and splits it into **workstreams**, by concern, the way you'd split tickets across a team. Workstreams can share files, since each implementer works in its own worktree; two workstreams never rewrite the same existing logic (adding side by side is fine), and each can be built without waiting on another one's unfinished code. Work that others depend on runs first, and they start as soon as it's merged.
2. It writes down the contracts where workstreams meet, sizes the team, has `plan-reviewer` check the breakdown (every file covered, shared files listed, no design collisions, contracts complete, each workstream bounded), and shows you the breakdown with the review's summary before spawning anyone.
3. Each workstream gets a **planner** teammate (Opus, high effort). The planner follows lets-do for its workstream and gives the plan a second correctness pass.
4. A **reviewer** teammate (Opus, high effort) then attacks each plan independently: wrong assumptions about the code, missed edge cases, contract breaks, vague structure or names, over-engineering. It works through graded findings with the planner in at most two rounds, and the lead settles anything still disputed. The lead then reviews all the plans together for consistency across workstreams.
5. Each workstream then gets an **implementer** teammate (medium effort). The implementer builds the plan in its own worktree, runs quality-suite on it and commits on its own branch.
6. The lead checks each branch against its plan (file map, design decisions, contracts), and merges the approved ones one at a time as they're ready, like a merge queue, testing each merge. A branch that conflicts or breaks the tests goes back to its implementer to rebase and fix, twice at most (trivial conflicts don't count). A workstream that can't be settled stays on its branch as **needs-human**, with the decision you need to make; the rest still merge. Pushing and opening a PR are left to you.

The planners, the plan reviewer and the implementers' quality-suite all apply lets-do's smallest-complete-change rules, and the report lists each workstream's cuts. "question the scope" makes the lead push back on parts of the task at the confirmation, before spawning anyone.

```
/dev-flow:team-do implement the plan above
/dev-flow:team-do docs/plans/billing.md
/dev-flow:team-do docs/plans/billing.md, 4 teammates
/dev-flow:team-do one workstream per module in the plan
/dev-flow:team-do add audit logging: frontend / backend / tests as workstreams, 3 teammates
/dev-flow:team-do the plan above; Sonnet for implementation, just go
/dev-flow:team-do the plan above, integrate on a separate branch
/dev-flow:team-do small refactor across three packages, no plan review
```

The argument is free text: the work plus any options, in plain words.

- **Team size**: "N teammates" caps how many workstreams run at once (each runs a planner, then an implementer, plus a reviewer while its plan is reviewed). Without it, the lead runs up to 5 at once and starts the rest as slots free. A workstream waits only for the workstreams it depends on, never for unrelated ones. If only one workstream can run at a time, it says a team isn't worth it and points you to lets-do.
- **Workstreams**: name them ("one per module", "frontend / backend / tests") and the lead uses yours instead of its own split.
- **Models**: name one per role ("Sonnet for implementation", "Opus for everything").
- **Effort** comes from the agent definitions in `agents/` and can't be changed per run. It applies to in-process teammates only; split-pane teammates use the lead's effort.
- **"no plan review"** drops the reviewer; plans go straight to the lead after the planner's second pass.
- **"just go"** skips the confirmation. **"integrate on a separate branch"** runs the merge queue in the lead's own worktree on a new `team/...` branch, so your checkout and branch are never touched; `git switch team/...` to see the result.

Requirements and tips:

- Agent teams are experimental: set `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` (shell or settings.json `env`) and run Claude Code interactively. Without it, team-do stops and tells you.
- Run the lead at high effort. Your session is the lead, and it does the highest-judgment work in the run: splitting the work, fixing the contracts, reviewing the plans together and checking each branch against its plan. Split-pane teammates also take the lead's effort instead of their agent's.
- Workstreams branch from your current commit, and by default the lead merges into your checkout, so commit or set aside changes to tracked files first (untracked files are fine). Untracked local files the tests need (an `.env`, local config) are copied into each worktree: list them in `.worktreeinclude`, or the lead asks. Worktrees go under `.claude/worktrees/`. On the default branch, the lead creates a `team/...` branch to integrate on.
- Teammate permission prompts show up in the lead's session: pre-approve common commands (tests, git worktree, git merge, git rebase, your build tool) before a long run.
- A team costs many times the tokens of one session; use it for work that genuinely splits.

## quality-suite

Runs the pre-merge chain on the current branch's changes, end to end, without stopping between steps. The steps, in order:

1. code-modularizer
2. `/simplify`
3. `/security-review`, only when the diff touches auth, untrusted input, shell, SQL, network or dependencies
4. `/code-review high --fix`
5. clean-code
6. independent verification: a `change-reviewer` agent that didn't write the code looks for what the chain missed
7. lint, typecheck and tests

Steps 1–5 each run again while they keep finding material issues, up to 3 passes, with fresh subagents for repeat passes and large diffs. Step 6 looks where a self-run, diff-focused review doesn't: claims in the plan, docs and copy against what the code does, tests that can't fail, effects on code the change didn't write, failure paths, duplication, and whether the findings the chain declined deserved it. You fix what it finds or dispute it; a second round runs when anything blocking or should-fix was fixed or disputed, and a blocking finding still held makes the run INCOMPLETE. It's skipped for trivial diffs or with "no verification". When it hands back, only commit and merge are left.

```
/dev-flow:quality-suite
/dev-flow:quality-suite only the last commit
/dev-flow:quality-suite against develop
/dev-flow:quality-suite what landed in a1b2c3d
/dev-flow:quality-suite small fix, no verification
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

### Lite, for small changes

`/dev-flow:quality-suite-lite` (or `/dev-flow:quality-suite lite`) runs the same chain with small-change settings:

| Step | Full | Lite |
|---|---|---|
| code-modularizer | always | only if a changed file is over the size limit |
| `/simplify`, `/code-review high --fix`, clean-code | repeat until they settle | one pass each |
| `/security-review` | when the diff calls for it | the same |
| independent verification | runs | skipped |
| lint, typecheck, tests, CLAUDE.md's gates | runs | the same |

Use it for small, contained changes: roughly under 100 changed lines, a handful of files, no new modules. On a bigger diff it says so and recommends the full suite, then runs what you asked for. When Claude starts quality-suite on its own, it always runs the full chain. lets-do can pick lite for a small plan; if the finished change turns out bigger than planned, that lite run switches to the full suite. Naming one in the task ("use quality-suite-lite", "full quality-suite") always wins, and a lite you chose stays lite. team-do's implementers run the full suite unless the task asks for lite.

```
/dev-flow:quality-suite-lite
/dev-flow:quality-suite-lite against next
/dev-flow:quality-suite lite
```

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

## The agents

dev-flow ships four agents in `agents/`:

| Agent | Model, effort | Job |
|---|---|---|
| `dev-flow:workstream-planner` | Opus, high | Plans one workstream with lets-do, second correctness pass, answers the reviewer |
| `dev-flow:plan-reviewer` | Opus, high | Attacks a plan independently, including anything it builds that the task doesn't need, and reports a verdict; used by lets-do and team-do |
| `dev-flow:change-reviewer` | Opus, high | Reviews a finished change for what the chain missed, including code that didn't need to exist; used by quality-suite |
| `dev-flow:workstream-implementer` | lead's model, medium | Builds the approved plan in its own worktree, runs quality-suite, commits on its branch, and rebases it when it conflicts |

The planner and implementer are written as teammates for team-do. Each expects a brief from a lead (goal, files with shared ones marked, contracts, base commit, the plugin's skills directory, paths) and reports back to that lead. You can still use them directly as subagents if you give them that brief yourself:

```
use the dev-flow:workstream-planner agent to plan the export module: its files are src/export/,
must not touch src/reports/, base commit HEAD, skills directory <path>/skills,
write the plan to /tmp/export-plan.md
```

lets-do runs the reviewer for you on every non-trivial plan. You can also point it at any plan directly; when its brief names no planner, it runs standalone and returns its graded findings and verdict to you.

```
use the dev-flow:plan-reviewer agent on /tmp/export-plan.md: goal is CSV export
from the reports page, its files: src/export/, base commit HEAD, skills directory <path>/skills
```

For planning in your own session, lets-do is usually the better fit. The planner is lets-do run for one workstream of a larger split, on Opus at high effort, without talking to you, with a second correctness pass and a review round required. The implementer has no skill counterpart.

## Credits

The smallest-complete-change rules in lets-do and the reviewers are adapted from [ponytail](https://github.com/DietrichGebert/ponytail) by Dietrich Gebert (MIT).

## License

MIT, see [LICENSE](LICENSE).
