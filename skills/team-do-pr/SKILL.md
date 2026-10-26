---
name: team-do-pr
description: Run a team-do agent team, then open one PR per workstream instead of one integrated branch - independent PRs, stacked only where a workstream depends on another, opened ready for review after a final confirmation. The lead rehearses the merge queue privately first, so the combined result is proven. Invoked as /dev-flow:team-do-pr followed by the task; /dev-flow:team-do-pr continue later rebases and restacks what your merges leave behind and says what's mergeable. Never merges a PR.
disable-model-invocation: true
argument-hint: "[task, plan path, or 'the plan above'] [options] | continue"
---

# Team do, one PR per workstream

The task: $ARGUMENTS

If the task starts with `continue`, go to **Continue**. Otherwise read `${CLAUDE_SKILL_DIR}/../team-do/SKILL.md` in full and lead it, with the differences below. If it can't be read, stop and say so.

Each workstream becomes its own PR, the way a team works: you review and merge on GitHub, in any order the stacking allows, and `continue` keeps the rest up to date. team-do's merge queue still runs, as a private rehearsal: it proves the workstreams work together before anything is published.

## Differences in team-do's steps

- **Step 0**, in addition:
  - The remote must be GitHub, `gh auth status` must pass, and git must be 2.38 or later (`git merge-tree --write-tree` needs it). If not, stop and say what to fix.
  - Resolve `origin`'s repository once, `gh repo view "$(git remote get-url origin)" --json nameWithOwner`, and pass `--repo <owner/name>` to every `gh` call in the run: with an `upstream` remote, `gh` would otherwise pick the wrong repository.
  - The PR base is the current branch. After `git fetch origin --prune`, `origin/<base>` must exist and `HEAD` must equal it: unpushed commits would land in every PR. If not, stop and ask the user to push or reset first.
  - Always integrate on a separate branch, in the integration worktree: it's a rehearsal and is never pushed.
- **Step 2**: a workstream depends on at most one other, since a PR has one base. A workstream that codes against a runtime contract another workstream writes (an endpoint, an event) depends on that writer, so its PR stacks on the writer's and can't merge first. If one would need two dependencies, fold them into one foundation workstream: B needing A and C becomes A+C, then B on it.
- **Step 4**: show each workstream's single dependency. Publishing is confirmed later, once the work exists.
- **Step 5**, at its start: workstream branch names must be free on `origin` too (`origin/team/<ws>`), not only locally.
- **Step 5**, throughout: tell every planner and implementer, in its brief: "your PR will hold only this workstream's commits, on `<base>` or on the one workstream you depend on; don't import, call or extend code from any other workstream, even if the code you read already has it". Add that to the plan check (5.1). When the review across workstreams settles a design collision, either way (a dependency, or a written runtime contract), the consumer depends on the other workstream, so only if it has no dependency yet; otherwise fold the two into one workstream. Record each workstream's merge commit, `M_<x>`.
- **Step 6** is replaced by the three parts below. Still ask every remaining teammate to shut down, keep NEEDS-HUMAN worktrees and branches, and clean up however the run ends, as team-do does.

## Build the PR branches

For each merged workstream, in merge order, rebuild its branch so it holds only its own commits. In the rehearsal those are exactly `M_<x>^1..team/<x>`, conflict fixes included.

- In a worktree on `team/<x>`: `git rebase --reapply-cherry-picks --onto <target> M_<x>^1 team/<x>`. The target is `origin/<base>`, or `team/<dep>` for its one dependency, already rebuilt since it merged earlier. `--reapply-cherry-picks` keeps a commit identical to another workstream's (the same lockfile regeneration, say).
- Then run the project's install, code generation, lint, typecheck and tests there. It's red only for failures not in team-do's baseline.
- A conflict: `git rebase --abort`, then hand the exact rebase command over. Red after a clean rebuild: hand over the failing output, and say explicitly not to rebase. Either way the handover goes to a `dev-flow:workstream-implementer` subagent, without a `name`, started at "When the lead sends your branch back" in subagent mode, with the worktree path, the plan path, the skills directory (the parent of this skill's base directory) and the target as its integration branch. On its report, start a `dev-flow:change-reviewer` subagent yourself on each resolution it didn't call mechanical. Send its blocking and should-fix findings to a fresh implementer subagent on the same worktree to fix or dispute, and decide disputes yourself; you don't edit the code. Two implementer hand-backs in all, then hold that workstream as NEEDS-HUMAN, with its dependents.

## Confirm and open

1. Before any push, show the user: each branch, its PR title and base, which PRs are stacked, any held workstream and why, and the rehearsal and rebuild results. Ask for a go-ahead with AskUserQuestion. Always ask, even when the task said "just go": this is what publishes the work. On a no, skip to step 4 and keep the rebuilt `team/*` branches and the rehearsal branch, listed in the report.
2. Push each PR branch: `git push -u origin team/<x>`.
3. Open each PR, ready for review: `gh pr create --repo <owner/name> --base <base, or team/<dep>> --head team/<x> --title <title> --body-file <file>`. Follow `.github/pull_request_template.md` if the repository has one. The body says:
   - the goal and what changed, the cuts (the plan's Not building), the quality-suite and plan-check results;
   - which PR it's stacked on, or that it's independent;
   - the plan, in a collapsed `<details>` block; leave it out, with a note, if the body would pass 60,000 characters;
   - last, a hidden marker: `<!-- team-do-pr run=<short-task-name>-<short base sha> base=<base> ws=<workstream> stacked-on=<parent workstream, or none> -->`. It's how `continue` finds the run.
4. Clean up, however the run ended: remove the worktrees (the integration worktree and every workstream's, except NEEDS-HUMAN ones); keep the local `team/*` branches. Once the PRs are open, delete the rehearsal integration branch (`git branch -D`: its work is in the PR branches), unless a workstream was held at Build: then first keep its proven rehearsal version as `team/<x>-rehearsal` (`git branch team/<x>-rehearsal M_<x>^2`), and list it in the report. Never merge into the base.

## Report

team-do's report, with these changes:
- The status line: "COMPLETE: N PRs open against `<base>`, ready for review", "PARTIAL: N PRs open, K need you", or "INCOMPLETE" with the blocker (a declined push included).
- Each workstream's PR link in its line.
- The suggested merge order: the rehearsal's order. Independent PRs can merge in any order. Merge a stacked PR only once its base shows `<base>`: after its parent merges, run `continue`, which retargets it. Merged while its base is still `team/<parent>`, it would land in the parent's branch, not in `<base>`.
- Last line: "After you merge any of them, run `/dev-flow:team-do-pr continue`."

## Continue

Run any time, in any session, after you merge or when a PR goes stale. It rebases and checks everything locally first, then asks once before it force-pushes or retargets anything. It never merges a PR and never pushes to the base.

1. **Find the run.** Resolve `origin`'s `<owner/name>` as in step 0, and pass `--repo` to every `gh` call below. `git fetch origin --prune`, then `gh pr list --state open --limit 500 --json number,title,headRefName,headRefOid,baseRefName,body,url,statusCheckRollup`, and keep the PRs whose head starts with `team/` and whose body has a `team-do-pr` marker. Filter locally, not with GitHub search. If several runs match, ask which. Take `<base>` from the marker. For each parent a marker names that isn't open, look it up: `gh pr list --head team/<parent> --state all --json number,state,mergedAt,headRefOid,baseRefName`.
   - A parent counts as merged only if it merged into `<base>`. One that merged into another `team/*` branch (merged while still stacked): report it as "merged into team/<its parent>, not <base>", and its children need you.
   - Note any closed without merging.
2. **Get ready.** For each open PR, `origin/team/<ws>` must equal its `headRefOid`; fetch again if not, and if it still differs, that PR needs you. For a merged parent whose branch is gone, `git fetch origin pull/<n>/head` to get its `headRefOid`. Find the untracked local files the checks need, as team-do's step 0 does (`.worktreeinclude`, or ask the user). Run the project's lint, typecheck and tests once on `origin/<base>` in a temporary worktree, as the baseline: from here on, checks are red only for new failures.
3. **Each open PR, parents before children.** If a local `team/<ws>` has commits `origin/team/<ws>` doesn't (`git rev-list origin/team/<ws>..team/<ws>`), that PR needs you: they're yours, unpushed. Otherwise record the sha of `origin/team/<ws>`, then `git worktree add --detach <path> origin/team/<ws>`, copy the untracked files in, and work there, on a detached HEAD: local branches move only after a push succeeds, so a no or a refused push leaves nothing behind. Decide its upstream and new base:
   - **Its parent PR merged** (stacked, the parent's state merged): upstream `git merge-base <parent's headRefOid> HEAD`, new base `origin/<base>`.
   - **Its parent was rebased in this run**: upstream `git merge-base <parent's recorded sha> HEAD`, new base the parent's rebased commit (its worktree's HEAD).
   - **Not stacked**: if `git merge-tree --write-tree origin/<base> HEAD` reports a conflict, or its CI is red, upstream `git merge-base origin/<base> HEAD`, new base `origin/<base>`. GitHub's own `mergeable` is often unknown right after a base moves, so don't rely on it.
   - **Stacked on an open parent that wasn't rebased in this run**: if `origin/team/<parent>` isn't an ancestor of `HEAD` (someone pushed to the parent, or an earlier restack stopped), that PR needs you, behind its parent. Otherwise leave it unchanged.
   - **Otherwise**: leave it unchanged.

   If `git rev-list --merges <upstream>..HEAD` isn't empty (a reviewer's "Update branch" merged into it, say), don't rebase: a rebase would drop that merge and its resolution, so that PR needs you. Otherwise `git rebase --reapply-cherry-picks --onto <new base> <upstream>`, then run the project's install, code generation, lint, typecheck and tests.
   - Clean and green: it's ready to push. Don't push yet.
   - A conflict or red checks: hand it to an implementer subagent and run the change-reviewer, as in Build. Its plan comes from the PR body's `<details>` block, written to a file; if the plan was left out, say so in the brief. Two rounds, then that PR needs you, and so do its children.
4. **Confirm.** If anything is ready, list it for the user: each branch to force-push, each PR to retarget, and what changed and why. Ask with AskUserQuestion; push nothing without a yes.
5. **Push, parents before children,** from each worktree: `git push --force-with-lease=team/<ws>:<recorded sha> origin HEAD:refs/heads/team/<ws>`. A refused lease means someone pushed to the branch meanwhile: that PR needs you, and don't push its children. After a push, move the local `team/<ws>` to it if there is one (`git branch -f team/<ws> HEAD`, from the main checkout if the branch isn't checked out), retarget the PR if its parent merged (`gh pr edit <n> --repo <owner/name> --base <base>`), and update its marker's `stacked-on`.
6. Remove the temporary worktrees.
7. **Report**: each PR as merged, updated (rebased and pushed, with CI pending), unchanged, or needs you (and why); every branch it force-pushed; then what's mergeable now: every PR based on `<base>` whose CI is green, plus the ones just pushed, as pending CI.
