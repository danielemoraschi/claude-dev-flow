---
name: team-do
description: Split a piece of work into parallel workstreams and run them with an agent team - you lead and coordinate; each workstream is planned by a teammate on Opus at high effort using lets-do, given a second correctness pass, then implemented by a teammate at medium effort in its own worktree and validated with quality-suite. Works out the workstreams and team size from the task or the plan already in the conversation. Invoked as /dev-flow:team-do followed by the task or a pointer to the plan; anything the user says in the task overrides these defaults.
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
- **Skip the confirmation** ("just go", "no confirmation"), **no worktrees** (implementers work in this checkout, on their own files only), or **leave the branches unmerged** (step 5's integrate becomes a report of the branches to merge).

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

- Each workstream has one teammate at a time: its planner, then its implementer. So the team size is the number of workstreams running at once.
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
- the absolute path of the lets-do skill file: `../lets-do/SKILL.md` from this skill's base directory, resolved

When a planner reports back, don't approve on its word. Once every plan in the wave is in, review them together: contracts match on both sides, no file appears in two workstreams, nothing falls between workstreams, steps are small and testable, and the plans embed clean-code and modularity decisions rather than just naming them. Send corrections to the planner that owns them and wait for the revision. Then ask the planners to shut down.

### Implement

Spawn one teammate per workstream with the `dev-flow:workstream-implementer` agent type, named `impl-<workstream>`. The spawn prompt carries:

- the plan file path, and the workstream's owned files and contracts
- the base commit, the worktree path (a sibling directory outside the checkout, e.g. `../<repo>-<workstream>`) and the branch to create (`team/<workstream>`)
- the names of the other implementers in the wave, for questions about contracts

While they work, answer their questions. When a teammate needs a decision that is genuinely the user's, ask the user and relay the answer. If a teammate stops early or fails, check its transcript, then redirect it or spawn a replacement for its workstream.

### Integrate

When every workstream in the wave has reported a committed, quality-suite-clean branch:

1. Merge the workstream branches into your branch in dependency order (`git merge --no-ff team/<workstream>`). Conflicts should be rare if ownership held; resolve them, or send them back to the workstream's implementer if they're more than mechanical.
2. Run the project's lint, typecheck and test commands on the merged result. Fix integration breakage, or send it to the workstream that owns the code.
3. Start the next wave from the new `HEAD`.

## 6. Finish

- If you wrote integration fixes yourself, run `/dev-flow:quality-suite` scoped to those fixes.
- Ask every remaining teammate to shut down.
- Remove each merged workstream's worktree (`git worktree remove`) and branch (`git branch -d`). Both refuse when work would be lost; if one refuses, leave it and say so.
- Don't push or open a pull request; that's the user's part.

## Report

End with a short summary. Its first line is the status: "COMPLETE, merged into <branch>, ready to review and push", or "INCOMPLETE" with what's pending and the blocker.

- waves and workstreams, with the teammates that planned and built each
- per workstream: plan file, quality-suite status, commits merged
- integration: conflicts resolved, fixes made, lint/typecheck/test results
- anything left unfixed, and any worktree or branch left behind
