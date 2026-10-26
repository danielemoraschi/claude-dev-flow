---
name: team-do
description: Split a piece of work into parallel workstreams and run them with an agent team - you lead and coordinate; each workstream is planned by a teammate on Opus at high effort using lets-do, given a second correctness pass and an independent adversarial review, then implemented by a teammate at medium effort in its own worktree and validated with quality-suite, then merged one at a time through a queue, rebasing on conflict. Works out the workstreams and team size from the task or the plan already in the conversation. Invoked as /dev-flow:team-do followed by the task or a pointer to the plan; anything the user says in the task overrides these defaults.
disable-model-invocation: true
argument-hint: "[task, plan path, or 'the plan above'] [options in plain words, e.g. 4 teammates]"
---

# Team do

The task: $ARGUMENTS

## Options in the task

The task is free text. Besides the work itself, it can set any of these in plain words, and what it says wins over the defaults below:

- **Team size**: "4 teammates" means at most 4 workstreams running at once (step 3). If you find fewer workstreams than that, use fewer and say so; if more, run them in waves of that size.
- **The workstreams**: if the task names them ("one per module in the plan", "frontend / backend / tests"), use those instead of drawing your own, still checking shared files, design collisions and contracts as in step 2.
- **Models**: "Sonnet for implementation" and similar. Name the model in the spawn prompt; that wins over the agent definition's model.
- **Effort** can't be changed per run: it comes from the agent definitions and is fixed at spawn. If the task asks for a different effort, say so in the confirmation.
- **Lite quality-suite** ("lite quality-suite for the implementers"): implementers run the full suite by default, because its independent reviewer is the only check on work no one reads before the merge. If the task asks for lite, pass that to the planners and implementers as the user's choice, and say in the confirmation that it skips the independent reviewer.
- **No plan review** ("no plan review", "skip the reviewer"): no review of your breakdown, and planners report straight to you after their second pass, with no reviewer teammate.
- **Question the scope** ("question the scope"): at step 4's confirmation, push back on any part of the task the need doesn't justify, with your recommendation, and plan only what the user confirms.
- **Skip the confirmation** ("just go", "no confirmation").
- **Integrate on a separate branch** ("separate branch", "don't touch my branch"): integrate on a new branch even when you're not on the default branch (step 0), and switch back to the original branch at the end (step 6).

You are the team lead. You find the workstreams, size the team, brief and review the teammates, and integrate their work. You don't implement workstreams yourself: wait for your teammates.

## 0. Check you can run a team

- Agent teams must be enabled: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in the environment or in settings.json `env`, in an interactive session. Check with `echo $CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS`. If it isn't set, stop and tell the user how to enable it; don't fall back to plain subagents silently.
- This must be a git repository: workstreams always work in their own worktrees.
- Worktrees go under `.claude/worktrees/` in the repository. Check it's ignored (`git check-ignore -q .claude/worktrees/` from the repository root); if it isn't, add `.claude/worktrees/` to `.git/info/exclude`, which stays local and is never committed, and say so.
- Tracked files must have no uncommitted changes (`git status --porcelain --untracked-files=no` empty): workstreams branch from the current `HEAD`, so uncommitted changes don't reach them, and you merge their work into this checkout. Untracked files are fine. If it isn't clean, ask the user to commit or set the changes aside; never stash or commit them yourself.
- Decide the integration branch: the current branch, unless it's the default branch (`main`, `master`, or what CLAUDE.md names) or the task says to integrate on a separate branch. Then it's a new `team/<short-task-name>`, created at the start of step 5, so a run that stops before then leaves the checkout as it was.

## 1. Find the work

The work comes from, in order: the task above (a description, a path to a plan or spec, an issue), and any plan already in this conversation. If there is neither, ask what to do before anything else.

If the work isn't broken down yet, read enough of the code to break it down. Don't write the detailed plan yourself; that's the planners' job. You need just enough to draw workstreams.

## 2. Draw the workstreams

A workstream is a self-contained unit with a clear deliverable that one teammate can plan and build without waiting on another workstream's unfinished code.

- **Split by concern.** One deliverable per workstream. Files can be shared: each implementer works in its own worktree, and overlaps are merged at integration.
- **List each workstream's files.** A file that several workstreams in the same wave change is a shared file: list what each one changes there, and keep those changes narrow. A file nearly every workstream changes heavily means the split is wrong, or that file needs a foundation workstream in an earlier wave.
- **No design collisions.** Two workstreams never change the behaviour of the same function, component or block: a textual conflict is cheap to merge, two designs for one piece of code aren't. If they would, merge them into one workstream or put them in successive waves.
- **Fix the contracts between workstreams now.** Where workstreams meet (a shared type, a function signature, an API shape, an event name), write the contract down before anyone starts. Exactly one workstream writes it; the others code against it. A contract consumers need to build or test against (a type, a function signature, an interface) must exist when they branch: if it isn't in the code yet, it goes in an earlier wave, as a small foundation workstream if need be. Only contracts met at runtime (an event name, an API shape) can be written in the same wave as their consumers, and those run one way: two workstreams never each code against a contract the other writes.
- **Sequence what can't run in parallel.** Work others build on (shared foundations, a schema, a core interface) goes in an earlier wave; workstreams in the next wave branch after it's merged. Items that must happen in order inside one area stay in one workstream.
- **Size workstreams sensibly.** A workstream of a few lines costs more to coordinate than it saves: fold it into a neighbor. A workstream that would take a teammate a very long time without a check-in: split it.

## 3. Size the team

- The team size is the number of workstreams running at once. Each workstream has one teammate at a time (its planner, then its implementer), plus a reviewer while its plan is under review, so planning briefly peaks at two teammates per workstream.
- Unless the task sets it, run every parallel workstream of a wave at once, up to 5 (the docs' 3-5 sweet spot: more adds coordination and tokens faster than speed). With more workstreams than that, run them in waves of 5.
- Shared files don't stop workstreams running in parallel; only dependencies and design collisions do. If no wave has two or more workstreams, a team adds cost without speed. Say so and suggest `/dev-flow:lets-do` instead, then stop.

## 4. Review the breakdown, then confirm

Your breakdown is the plan every teammate works inside, so have it attacked before anyone builds on it. Skip this only when the task says "no plan review".

1. Start the `dev-flow:plan-reviewer` agent as a subagent, without a `name` (a named one would launch as a teammate). Its brief: the task, your breakdown (waves; each workstream's goal, files and scope; the shared files with what each workstream changes; the contracts; anything cut), the skills directory (the parent of this skill's base directory, resolved), the base commit, and that it's a team-do breakdown. If that agent type isn't available in this session, start an unnamed general-purpose subagent instead, told to read `${CLAUDE_SKILL_DIR}/../../agents/plan-reviewer.md` and act as that agent, with the same brief. If neither is possible, say the review was skipped and why.
2. Fix each finding in the breakdown, or dispute it with a reason grounded in the code. If any blocking finding was fixed or disputed, start a fresh reviewer with the earlier findings and your responses. Two rounds at most.
3. Decisions the reviewer flags as the user's go into the confirmation as questions.

A team spends many times the tokens of one session, so show the user the breakdown once before spawning anyone: the waves, each workstream (name, goal, files with shared ones marked, depends on), the contracts, the team size, the models and effort, and a short summary of the breakdown review (rounds, findings by grade, what changed). Ask for a go-ahead (with AskUserQuestion if available), and adjust if the user changes anything. Skip this only if the task says to go without confirming.

## 5. Run each wave

If step 0 decided on a new integration branch, create and check it out now (`git switch -c team/<short-task-name>`), with a name that doesn't exist yet and isn't a workstream's.

Pick a directory outside the repository for plans (your scratchpad if you have one, otherwise a temp directory), and record the base commit the wave starts from (`git rev-parse HEAD`).

### Plan

Spawn one teammate per workstream with the `dev-flow:workstream-planner` agent type, named `plan-<workstream>`. Pass a `name` and no `isolation`: a call with `isolation` starts a plain subagent, not a teammate. Teammates don't see this conversation, so the spawn prompt carries everything:

- the workstream: goal, scope, its files, the shared files with what every workstream changes in each, files it must not touch, and why the workstream exists
- the contracts with other workstreams, word for word, and which teammate owns each other workstream
- the base commit, and the path of the plan file to write in the plans directory
- the plugin's skills directory, as an absolute path: the parent of this skill's base directory, resolved (it holds `lets-do/`, `clean-code/` and `code-modularizer/`)

### Review each plan

When a planner says its plan is ready for review, spawn a `dev-flow:plan-reviewer` teammate for it, named `review-<workstream>`. Give it the same brief and skills directory as the planner, the plan file path, and the planner's name. Don't pass it your own opinion of the plan or the planner's reasoning: it reviews independently. The reviewer and planner work through the findings directly, in at most two rounds, and the reviewer sends you a verdict: approved, or disputed with the open blocking findings and both positions. Decide each disputed finding yourself, or ask the user when it's genuinely theirs, and tell the planner. Then ask the reviewer to shut down.

### Review across workstreams

Once every plan in the wave is reviewed and its planner has reported, review the plans together, for what no single reviewer could see:

- **Contracts** match on both sides, and nothing falls between workstreams.
- **Shared files**: each plan's changes match the breakdown, and no two plans add the same helper.
- **Design collisions**: no two plans change the same function, component or block. Settle one either by moving one workstream to the next wave, where it's planned again from that wave's base with its old plan as input and reviewed as usual, or, when one side only needs the other's runtime contract (an event name, an API shape), by writing that contract down and re-briefing both planners.
- **Cuts**: no plan's Not building list cuts something the task asked for, and two workstreams' cuts don't leave a gap between them. A cut that drops something the user asked for goes back to its planner.
- **Shape**: spot-check that each plan has the lets-do shape: design decisions tied to clean-code and code-modularizer rules and carried by the steps, a measured file map, and structure, code, reuses, errors and tests in every step.

Send corrections to the planner that owns them and wait for the revision. Then ask the planners to shut down.

### Implement

Spawn one teammate per workstream with the `dev-flow:workstream-implementer` agent type, named `impl-<workstream>`. The spawn prompt carries:

- the plan file path, the workstream's files, the shared files with what every workstream changes in each, and the contracts
- the base commit, the worktree path (`<repository root>/.claude/worktrees/team-<workstream>`, as an absolute path), the branch to create (`team/<workstream>`) and the integration branch's name
- the plugin's skills directory and the plans directory, for reviewing a rebase
- the names of the other implementers in the wave, for questions about contracts

Implementers stay available until you tell them the wave is done. While they work, answer their questions. When a teammate needs a decision that is genuinely the user's, ask the user and relay the answer. If a teammate stops early or fails, check its transcript, then redirect it or spawn a replacement for its workstream.

### Integrate as a queue

Work like a team's merge queue: decide and merge each workstream as its implementer reports, without waiting for the rest of the wave.

1. **Check it against its plan.** Compare the branch's diff (`git diff <integration branch>...team/<workstream>`, which stays right after a rebase) with the approved plan: every file in the file map and only those, each design decision carried out where the plan put it, and the contracts with other workstreams exactly as briefed. Send gaps back to the implementer, which is still running, for one round of fixes, then check again.
2. **Approve or escalate.**
   - **APPROVED**: quality-suite ended COMPLETE and the plan check passes.
   - **NEEDS-HUMAN**: quality-suite ended INCOMPLETE, the plan check still fails after the round of fixes, or the workstream raised a decision that's genuinely the user's. Leave its branch unmerged and keep its worktree.
   - A workstream that depends on a NEEDS-HUMAN one, through a contract in this wave or as a later wave, is held too, as NEEDS-HUMAN (blocked by it). Everything else carries on.
3. **Queue it.** Approved branches merge in the order they were approved, except that a branch coding against a contract another workstream in the wave writes waits until that writer is merged. Before the first merge of the run, run the project's lint, typecheck and test commands once on the integration branch as the baseline. Report any failures already there to the user and carry on: from then on, a merge is red only for new failures.
4. **Merge the next branch in the queue.** A fix commit queued for a waiting branch goes first. Otherwise skip any branch whose changed files (`git diff --name-only <integration branch>...team/<workstream>`) overlap another branch that's out (sent back and not yet re-queued): merging it first would only make the returning branch conflict again.
   - Check `git status --porcelain --untracked-files=no` is empty (ask the user if it isn't), then `git merge --no-ff --no-commit team/<workstream>`. If it merged without conflicts, run lint in check mode (no autofix), typecheck and tests, in CI mode where the project has one. Rerun a single new failure once before calling it red: it may be flaky.
   - If the checks changed tracked files (`git diff --name-only` lists them: snapshots, generated files), discard those changes with `git checkout -- .`, which keeps the staged merge, and count the merge as red, with those files as the failing output.
   - Green: `git commit --no-edit`.
   - A conflict, or red against the baseline: list the conflicting files (`git diff --name-only --diff-filter=U`), `git merge --abort`, note the branch's tip, and send it back to its implementer to rebase onto the integration branch and fix, with the conflicting files or the failing output, and the workstreams it conflicts with and their plan paths.
   - If the fix belongs in a workstream merged earlier in this wave, send it to that workstream's implementer for a fix commit on its branch, rebased onto the integration branch, and queue that next. The failing branch waits for it; the round counts against the failing branch. If it belongs in an earlier wave's workstream, the failing branch's implementer makes the fix on its own branch, once you've checked no other workstream in this wave changes that code.
5. **When a branch comes back,** read `git range-diff --creation-factor=100 <integration branch> <old tip> team/<workstream>`: it shows exactly what changed from the branch you approved, as a diff of each commit's patch. A resolution called mechanical that edits either side's logic goes back for a change-reviewer pass; that send-back keeps the branch out but isn't a round. Decide each blocking finding the implementer disputed, as for plan reviews; one that stands sends the branch back as its next round. A branch with no open blocking finding rejoins the queue at its original approval position.
6. **Two rebase rounds per workstream,** then NEEDS-HUMAN, keeping its branch and worktree. A round whose resolution was mechanical and needed no fix commit doesn't count: trivial conflicts in shared files never push a branch to NEEDS-HUMAN.
7. **Next wave.** The wave is done when every workstream is merged or NEEDS-HUMAN. Ask its implementers to shut down, then start the next wave from the new `HEAD`, without the held workstreams.

You don't implement here either: conflicts and integration breakage go back to the implementers.

## 6. Finish

Do this however the run ends once step 5 has started, including a run that merged nothing.

- Ask every remaining teammate to shut down.
- With the integration branch still checked out (`git branch -d` checks against it), remove each merged workstream's worktree (`git worktree remove`) and branch (`git branch -d`). Both refuse when work would be lost; if one refuses, leave it and say so. Leave NEEDS-HUMAN workstreams' worktrees and branches in place for the user.
- If the task said to integrate on a separate branch, switch back to the original branch last.
- Don't push or open a pull request; that's the user's part.

## Report

End with a short summary. Its first line is the status: "COMPLETE, merged into <branch>, ready to review and push", "PARTIAL: N of M merged, K need you", or "INCOMPLETE" with the blocker when the run couldn't start or merged nothing.

- **needs you**, first: per NEEDS-HUMAN workstream, its branch and worktree, why it was held, the open findings, and the decision to make, with your recommendation
- **the breakdown review**, always: rounds, findings by grade, what it changed, or that it was skipped and why
- **cuts**, always: each workstream's Not building items, one line each, copied from its plan, so the user sees every cut without opening the plans; "none" when a plan cut nothing
- waves and workstreams, with the teammates that planned, reviewed and built each
- per workstream: APPROVED or NEEDS-HUMAN, plan file, review verdict and how disputes were settled, quality-suite status, plan-check result, commits merged
- integration: the merge order; per workstream, clean or rebased (rounds, resolution mechanical or reviewed); the baseline and each merge's lint/typecheck/test results
- anything left unfixed, and any worktree or branch left behind
