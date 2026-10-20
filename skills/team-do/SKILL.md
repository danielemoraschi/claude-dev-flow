---
name: team-do
description: Split a piece of work into parallel workstreams and run them with an agent team - you lead and coordinate; each workstream is planned by a teammate on Opus at high effort using lets-do, given a second correctness pass and an independent adversarial review, then implemented by a teammate at medium effort in its own worktree and validated with quality-suite. Works out the workstreams and team size from the task or the plan already in the conversation. Invoked as /dev-flow:team-do followed by the task or a pointer to the plan; anything the user says in the task overrides these defaults.
disable-model-invocation: true
argument-hint: "[task, plan path, or 'the plan above'] [options in plain words, e.g. 4 teammates]"
---

# Team do

The task: $ARGUMENTS

## Options in the task

The task is free text. Besides the work itself, it can set any of these in plain words, and what it says wins over the defaults below:

- **Team size**: "4 teammates" means at most 4 workstreams running at once (step 3). If you find fewer workstreams than that, use fewer and say so; if more, run them in waves of that size.
- **The workstreams**: if the task names them ("one per module in the plan", "frontend / backend / tests"), use those instead of drawing your own, still checking file ownership and contracts as in step 2.
- **Models**: "Sonnet for implementation" and similar. Name the model in the spawn prompt; that wins over the agent definition's model.
- **Effort** can't be changed per run: it comes from the agent definitions and is fixed at spawn. If the task asks for a different effort, say so in the confirmation.
- **No plan review** ("no plan review", "skip the reviewer"): planners report straight to you after their second pass, with no reviewer teammate.
- **Skip the confirmation** ("just go", "no confirmation"), **no worktrees** (implementers work in this checkout, on their own files only), or **leave the branches unmerged** (step 5's integrate still checks and decides each workstream, then lists the approved branches instead of merging them).

You are the team lead. You find the workstreams, size the team, brief and review the teammates, and integrate their work. You don't implement workstreams yourself: wait for your teammates.

## 0. Check you can run a team

- Agent teams must be enabled: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in the environment or in settings.json `env`, in an interactive session. Check with `echo $CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`. If it isn't set, stop and tell the user how to enable it; don't fall back to plain subagents silently.
- This must be a git repository: workstreams work in their own worktrees. Workstreams branch from the current `HEAD`, so uncommitted changes don't reach them. If the work depends on uncommitted changes, ask the user to commit them first; don't commit them yourself.
- If the current branch is the default branch (`main`, `master`, or what CLAUDE.md names), create a branch for the work (`team/<short-task-name>`) and integrate there.

## 1. Find the work

The work comes from, in order: the task above (a description, a path to a plan or spec, an issue), and any plan already in this conversation. If there is neither, ask what to do before anything else.

If the work isn't broken down yet, read enough of the code to break it down. Don't write the detailed plan yourself; that's the planners' job. You need just enough to draw workstreams.

## 2. Draw the workstreams

A workstream is a self-contained unit with a clear deliverable that one teammate can plan and build without waiting on another workstream's unfinished code.

- **Each workstream owns its files.** No file belongs to two workstreams; two teammates editing the same file overwrite each other. If two items must touch the same file, they're one workstream.
- **Fix the contracts between workstreams now.** Where workstreams meet (a shared type, a function signature, an API shape, an event name), write the contract down before anyone starts, and give the file that holds it to exactly one workstream.
- **Sequence what can't run in parallel.** Work others build on (shared foundations, a schema, a core interface) goes in an earlier wave; workstreams in the next wave branch after it's merged. Items that must happen in order inside one area stay in one workstream.
- **Size workstreams sensibly.** A workstream of a few lines costs more to coordinate than it saves: fold it into a neighbor. A workstream that would take a teammate a very long time without a check-in: split it.

## 3. Size the team

- The team size is the number of workstreams running at once. Each workstream has one teammate at a time (its planner, then its implementer), plus a reviewer while its plan is under review, so planning briefly peaks at two teammates per workstream.
- Unless the task sets it, run every parallel workstream of a wave at once, up to 5 (the docs' 3-5 sweet spot: more adds coordination and tokens faster than speed). With more workstreams than that, run them in waves of 5.
- If no wave has two or more workstreams, a team adds cost without speed. Say so and suggest `/dev-flow:lets-do` instead, then stop.

## 4. Confirm before spawning

A team spends many times the tokens of one session, so show the user the breakdown once before spawning anyone: the waves, each workstream (name, goal, owned files, depends on), the contracts, the team size, and the models and effort. Ask for a go-ahead (with AskUserQuestion if available), and adjust if the user changes anything. Skip this only if the task says to go without confirming.

## 5. Run each wave

Pick a directory outside the repository for plans (your scratchpad if you have one, otherwise a temp directory), and record the base commit the wave starts from (`git rev-parse HEAD`).

### Plan

Spawn one teammate per workstream with the `dev-flow:workstream-planner` agent type, named `plan-<workstream>`. Pass a `name` and no `isolation`: a call with `isolation` starts a plain subagent, not a teammate. Teammates don't see this conversation, so the spawn prompt carries everything:

- the workstream: goal, scope, owned files, files it must not touch, and why the workstream exists
- the contracts with other workstreams, word for word, and which teammate owns each other workstream
- the base commit, and the path of the plan file to write in the plans directory
- the plugin's skills directory, as an absolute path: the parent of this skill's base directory, resolved (it holds `lets-do/`, `clean-code/` and `code-modularizer/`)

### Review each plan

When a planner says its plan is ready for review, spawn a `dev-flow:workstream-plan-reviewer` teammate for it, named `review-<workstream>`. Give it the same brief and skills directory as the planner, the plan file path, and the planner's name. Don't pass it your own opinion of the plan or the planner's reasoning: it reviews independently. The reviewer and planner work through the findings directly, in at most two rounds, and the reviewer sends you a verdict: approved, or disputed with the open blocking findings and both positions. Decide each disputed finding yourself, or ask the user when it's genuinely theirs, and tell the planner. Then ask the reviewer to shut down.

### Review across workstreams

Once every plan in the wave is reviewed and its planner has reported, review the plans together, for what no single reviewer could see: contracts match on both sides, no file appears in two workstreams, and nothing falls between workstreams. Spot-check that each plan has the lets-do shape: design decisions tied to clean-code and code-modularizer rules and carried by the steps, a measured file map, and structure, code, errors and tests in every step. Send corrections to the planner that owns them and wait for the revision. Then ask the planners to shut down.

### Implement

Spawn one teammate per workstream with the `dev-flow:workstream-implementer` agent type, named `impl-<workstream>`. The spawn prompt carries:

- the plan file path, and the workstream's owned files and contracts
- the base commit, the worktree path (a sibling directory outside the checkout, e.g. `../<repo>-<workstream>`) and the branch to create (`team/<workstream>`)
- the names of the other implementers in the wave, for questions about contracts

While they work, answer their questions. When a teammate needs a decision that is genuinely the user's, ask the user and relay the answer. If a teammate stops early or fails, check its transcript, then redirect it or spawn a replacement for its workstream.

### Integrate

When every implementer in the wave has reported, decide each workstream:

1. **Check it against its plan.** Compare the branch's diff (`git diff <base commit>...team/<workstream>`) with the approved plan: every file in the file map and only those, each design decision carried out where the plan put it, and the contracts with other workstreams exactly as briefed. Send gaps back to the implementer, which is still running, for one round of fixes, then check again.
2. **Approve or escalate.**
   - **APPROVED**: quality-suite ended COMPLETE and the plan check passes.
   - **NEEDS-HUMAN**: quality-suite ended INCOMPLETE, the plan check still fails after the round of fixes, or the workstream raised a decision that's genuinely the user's. Leave its branch unmerged and keep its worktree.
   - A workstream in a later wave that depends on a NEEDS-HUMAN one is held too, as NEEDS-HUMAN (blocked by it). Everything else carries on.

Then, for the approved workstreams:

3. Merge their branches into your branch in dependency order (`git merge --no-ff team/<workstream>`). Conflicts should be rare if ownership held; resolve them, or send them back to the workstream's implementer if they're more than mechanical.
4. Run the project's lint, typecheck and test commands on the merged result. Fix integration breakage, or send it to the workstream that owns the code.
5. Start the next wave from the new `HEAD`, without the held workstreams.

## 6. Finish

- If you wrote integration fixes yourself, run `/dev-flow:quality-suite` scoped to those fixes.
- Ask every remaining teammate to shut down.
- Remove each merged workstream's worktree (`git worktree remove`) and branch (`git branch -d`). Both refuse when work would be lost; if one refuses, leave it and say so. Leave NEEDS-HUMAN workstreams' worktrees and branches in place for the user.
- Don't push or open a pull request; that's the user's part.

## Report

End with a short summary. Its first line is the status: "COMPLETE, merged into <branch>, ready to review and push", "PARTIAL: N of M merged, K need you", or "INCOMPLETE" with the blocker when the run couldn't start or merged nothing.

- **needs you**, first: per NEEDS-HUMAN workstream, its branch and worktree, why it was held, the open findings, and the decision to make, with your recommendation
- waves and workstreams, with the teammates that planned, reviewed and built each
- per workstream: APPROVED or NEEDS-HUMAN, plan file, review verdict and how disputes were settled, quality-suite status, plan-check result, commits merged
- integration: conflicts resolved, fixes made, lint/typecheck/test results
- anything left unfixed, and any worktree or branch left behind
