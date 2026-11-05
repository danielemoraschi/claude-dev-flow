---
name: team-do-pr
description: Run a team-do agent team, then open one PR per workstream instead of one integrated branch - workstreams whose changes conflict, or that depend on each other, become a stack carrying the rehearsal's conflict fixes; the rest are independent PRs ("independent PRs" opts out of stacking). Opened ready for review after a final confirmation. The lead rehearses the merge queue privately first, so the combined result is proven. Invoked as /dev-flow:team-do-pr followed by the task; /dev-flow:team-do-pr continue later rebases, retargets and restacks what your merges leave behind and says what's ready. Never merges a PR.
disable-model-invocation: true
argument-hint: "[task, plan path, or 'the plan above'] [options] | continue"
---

# Team do, one PR per workstream

The task: $ARGUMENTS

If the task starts with `continue`, go to **Continue**. Otherwise read `${CLAUDE_SKILL_DIR}/../team-do/SKILL.md` in full and lead it, with the differences below. If it can't be read, stop and say so.

Each workstream becomes its own PR, the way a team works: you review and merge on GitHub, and `continue` keeps the rest up to date. team-do's merge queue still runs, as a private rehearsal: it proves the workstreams work together, and resolves their conflicts once, before anything is published. Workstreams whose changes conflict, or that depend on each other, become a **stack** (each PR based on the one below it, carrying the rehearsal's fixes); the rest are independent PRs on the base.

**Options:** team-do's, plus **independent PRs** ("independent PRs", "don't stack"): every PR on the base, stacked only on a single declared dependency, and conflicts between them resolved again at rebuild and by `continue`. Choose it when you need to merge overlapping work in any order.

## Differences in team-do's steps

- **Step 0**, in addition:
  - The remote must be GitHub, `gh auth status` must pass, and git must be 2.40 or later (`git merge-tree --merge-base` needs it). If not, stop and say what to fix.
  - Resolve `origin`'s repository once, `gh repo view "$(git remote get-url origin)" --json nameWithOwner`, and pass `--repo <owner/name>` to every `gh` call in the run: with an `upstream` remote, `gh` would otherwise pick the wrong repository.
  - The PR base is the current branch. After `git fetch origin --prune`, `origin/<base>` must exist and `HEAD` must equal it: unpushed commits would land in every PR. If not, stop and ask the user to push or reset first.
  - Always integrate on a separate branch, in the integration worktree: it's a rehearsal and is never pushed.
- **Step 2**: a workstream that codes against a runtime contract another workstream writes (an endpoint, an event) depends on that writer. With "independent PRs" only, a workstream depends on at most one other, since its PR has one base; fold two dependencies into one foundation workstream (B needing A and C becomes A+C, then B on it).
- **Step 4**: show each workstream's dependencies and the stacks you expect (from dependencies, and from shared files likely to conflict), saying the final stacks come from the rehearsal. Publishing is confirmed later, once the work exists.
- **Step 5**, at its start: workstream branch names must be free on `origin` too (`origin/team/<ws>`), not only locally.
- **Step 5**, throughout: tell every planner and implementer, in its brief: "your PR may be rebuilt on `<base>`, or above only the workstreams it depends on or conflicts with; don't import, call or extend code from a workstream you don't depend on, even if the code you read already has it". Add that to the plan check (5.1). Design collisions are settled as team-do says; with "independent PRs", settle one with a dependency only if the consumer has none yet, otherwise fold the two into one workstream. Record each workstream's merge commit, `M_<x>`.
- **Step 6** is replaced by the three parts below. Still ask every remaining teammate to shut down, keep NEEDS-HUMAN worktrees and branches, and clean up however the run ends, as team-do does.

## Build the PR branches

**Link and stack** (skip with "independent PRs"). For each merged workstream Y, in merge order:
- Find where its own changes conflict without the others: `git merge-tree --write-tree --name-only --merge-base M_Y^1 origin/<base> team/Y` lists those files.
- Y links to every earlier workstream that changed one of them. A workstream's changed files are `git log --no-renames --name-only --format= M_X^1..M_X^2` (every commit, both sides of a rename).
- Y also links to every workstream it depends on.

Each connected group, in merge order, is one stack: a straight line, so if C links to both A and E, all three are one stack. A workstream with no links is an independent PR.

**Rebuild**, in merge order, so each branch holds only its own commits (in the rehearsal, exactly `M_<x>^1..team/<x>`, conflict fixes included):
- In a worktree on `team/<x>`, start from its rehearsal tip, `git reset --hard M_<x>^2`, so a re-run never replays an earlier rebuild; then `git rebase --reapply-cherry-picks --onto <target> M_<x>^1 team/<x>`. The target is the previous member of its stack, already rebuilt, or `origin/<base>` for the first member and for an independent PR. With "independent PRs", the target is `origin/<base>` or `team/<dep>` for its one dependency. `--reapply-cherry-picks` keeps a commit identical to another workstream's (the same lockfile regeneration, say).
- Then run the project's install, code generation, lint, typecheck and tests there. It's red only for failures not in team-do's baseline.
- A conflict: `git rebase --abort`, then hand the exact rebase command over. Red after a clean rebuild: hand over the failing output, and say explicitly not to rebase. Either way the handover goes to a `dev-flow:workstream-implementer` subagent, without a `name`, started at "When the lead sends your branch back" in subagent mode, with the worktree path, the plan path, the skills directory (the parent of this skill's base directory) and the target as its integration branch. On its report, start a `dev-flow:change-reviewer` subagent yourself on each resolution it didn't call mechanical. Send its blocking and should-fix findings to a fresh implementer subagent on the same worktree to fix or dispute, and decide disputes yourself; you don't edit the code. Two implementer hand-backs in all, then hold that workstream as NEEDS-HUMAN.
- **A held workstream stops its stack**: the members below it are opened; it and every member above it are held as NEEDS-HUMAN (blocked by it), their `team/*` branches left as the rehearsal made them, not rebuilt (`git branch -f team/<x> M_<x>^2` for any rebuilt in an earlier pass). Its dependents are held in either mode.

## Confirm and open

1. Before any push, show the user: each stack (its members in order, their bases, and the files or dependencies that link them), the independent PRs, any held workstream and why, and the rehearsal and rebuild results. Say that stacked PRs may show no CI until they target `<base>` (many repositories run CI only on PRs into it), so the local checks are their proof until then. Ask for a go-ahead with AskUserQuestion. Always ask, even when the task said "just go": this is what publishes the work.
   - If any stack was formed by conflicts and not only dependencies, offer a third answer, **stack only on dependencies**: rebuild with dependency links only, so workstreams linked just by conflicts become independent PRs (their conflicts resolved again at rebuild and later by `continue`). Then show this confirmation again.
   - On a no, skip to step 4 and keep the rebuilt `team/*` branches and the rehearsal branch, listed in the report.
2. Push each PR branch: `git push -u origin team/<x>`.
3. Open each PR, ready for review: `gh pr create --repo <owner/name> --base <base, or team/<previous member> (its dependency, with independent PRs)> --head team/<x> --title <title> --body-file <file>`. Follow `.github/pull_request_template.md` if the repository has one. The body says:
   - first, a status line: "**Stack: waiting on #N.** Merge this only when this line says Ready; `/dev-flow:team-do-pr continue` updates it." for a stacked PR above the first, "**Ready**" for a stack's first PR, or "**Ready** (independent)". For a stacked PR, add that its CI may not run until it targets `<base>`;
   - the goal and what changed, the cuts (the plan's Not building), the quality-suite and plan-check results;
   - the plan, in a collapsed `<details>` block; leave it out, with a note, if the body would pass 60,000 characters;
   - last, a hidden marker: `<!-- team-do-pr run=<short-task-name>-<short base sha> base=<base> ws=<workstream> stacked-on=<previous member, or its dependency with independent PRs, or none> -->`. It's how `continue` finds the run.
4. Clean up, however the run ended: remove the worktrees (the integration worktree and every workstream's, except NEEDS-HUMAN ones); keep the local `team/*` branches. Once the PRs are open, delete the rehearsal integration branch (`git branch -D`: its work is in the PR branches), unless a workstream was held at Build: then keep it, since it's the only record of where each held workstream's own commits start, and list it in the report with each held workstream's `M_<x>^1` and `M_<x>^2` and the rebuild command. Never merge into the base.

## Report

team-do's report, with these changes:
- The status line: "COMPLETE: N PRs open against `<base>`, ready for review", "PARTIAL: N PRs open, K need you", or "INCOMPLETE" with the blocker (a declined push included).
- Each workstream's PR link in its line, grouped by stack.
- How to merge: each stack bottom-up, and a PR only once its status line says Ready (run `continue` after each merge to update the next one); stacks and independent PRs in any order relative to each other. GitHub may retarget a stacked PR to `<base>` on its own when its parent's branch is deleted; that doesn't make it ready.
- Last line: "After you merge any of them, run `/dev-flow:team-do-pr continue`."

## Continue

Run any time, in any session, after you merge or when a PR goes stale. It rebases and checks everything locally first, then asks once before it force-pushes or retargets anything. It never merges a PR and never pushes to the base.

1. **Find the run.** Resolve `origin`'s `<owner/name>` as in step 0, and pass `--repo` to every `gh` call below. `git fetch origin --prune`, then `gh pr list --state open --limit 500 --json number,title,headRefName,headRefOid,baseRefName,body,url,statusCheckRollup`, and keep the PRs whose head starts with `team/` and whose body has a `team-do-pr` marker. Filter locally, not with GitHub search. If several runs match, ask which. Take `<base>` from the marker. For each parent a marker names that isn't open, look it up: `gh pr list --head team/<parent> --state all --json number,state,mergedAt,headRefOid,baseRefName`.
   - A parent counts as merged only if it merged into `<base>`. One that merged into another `team/*` branch (merged while still stacked), or was closed without merging: report it, and its children need you.
2. **Get ready.** For each open PR, `origin/team/<ws>` must equal its `headRefOid`; fetch again if not, and if it still differs, that PR needs you. For a merged parent whose branch is gone, `git fetch origin pull/<n>/head` to get its `headRefOid`. Find the untracked local files the checks need, as team-do's step 0 does (`.worktreeinclude`, or ask the user). Run the project's lint, typecheck and tests once on `origin/<base>` in a temporary worktree, as the baseline: from here on, checks are red only for new failures.
3. **Each open PR, parents before children.** If a local `team/<ws>` has commits `origin/team/<ws>` doesn't (`git rev-list origin/team/<ws>..team/<ws>`), that PR needs you: they're yours, unpushed. Otherwise record the sha of `origin/team/<ws>`, then `git worktree add --detach <path> origin/team/<ws>`, copy the untracked files in, and work there, on a detached HEAD: local branches move only after a push succeeds, so a no or a refused push leaves nothing behind. Decide what it needs:
   - **Its parent PR merged into `<base>`**: if `git merge-base --is-ancestor <parent's headRefOid> origin/<base>` (a merge-commit merge) and `git merge-tree --write-tree origin/<base> HEAD` reports no conflict, it only needs retargeting, with no rebase: then close and reopen it, so CI runs against the base (a retarget alone doesn't start CI, and its old checks tested the old base). Otherwise (a squash or rebase merge, or a conflict), upstream `git merge-base <parent's headRefOid> HEAD`, new base `origin/<base>`.
   - **Its parent was rebased in this run**: upstream `git merge-base <parent's recorded sha> HEAD`, new base the parent's rebased commit (its worktree's HEAD).
   - **Not stacked**: if `git merge-tree --write-tree origin/<base> HEAD` reports a conflict, or its CI is red, upstream `git merge-base origin/<base> HEAD`, new base `origin/<base>`. GitHub's own `mergeable` is often unknown right after a base moves, so don't rely on it.
   - **Stacked on an open parent that wasn't rebased in this run**: if `origin/team/<parent>` isn't an ancestor of `HEAD` (someone pushed to the parent, or an earlier restack stopped), that PR needs you, behind its parent. Otherwise leave it unchanged.
   - **Otherwise**: leave it unchanged.

   To rebase: if `git rev-list --merges <upstream>..HEAD` isn't empty (a reviewer's "Update branch" merged into it, say), don't: a rebase would drop that merge and its resolution, so that PR needs you. Otherwise `git rebase --reapply-cherry-picks --onto <new base> <upstream>`, then run the project's install, code generation, lint, typecheck and tests.
   - Clean and green: it's ready to push. Don't push yet.
   - A conflict or red checks: hand it to an implementer subagent and run the change-reviewer, as in Build. Its plan comes from the PR body's `<details>` block, written to a file; if the plan was left out, say so in the brief. Two rounds, then that PR needs you, and so do its children.
4. **Confirm.** If anything is ready, list it for the user: each branch to force-push, each PR to retarget (and close and reopen, for one only retargeted), and what changed and why. Ask with AskUserQuestion; push nothing without a yes.
5. **Apply, parents before children.**
   - If its parent merged into `<base>`, retarget it first (`gh pr edit <n> --repo <owner/name> --base <base>`), so the push below triggers CI against the base.
   - If it was rebased, push from its worktree: `git push --force-with-lease=team/<ws>:<recorded sha> origin HEAD:refs/heads/team/<ws>`. A refused lease means someone pushed to the branch meanwhile: that PR needs you, and don't push its children. After the push, move the local `team/<ws>` to it if there is one (`git branch -f team/<ws> HEAD`, from the main checkout if the branch isn't checked out).
   - If it was only retargeted, close and reopen it (`gh pr close <n> --repo <owner/name>`, then `gh pr reopen <n> --repo <owner/name>`): that starts CI against `<base>`.
   - Update its body: the status line and the marker's `stacked-on`. The line says "**Ready**" once it targets `<base>` and was pushed or reopened in this run (CI then runs against the base); children still "waiting on #N".
6. Remove the temporary worktrees.
7. **Report**: each PR as merged, updated (rebased and pushed, with CI pending), retargeted (closed and reopened to start CI against `<base>`, CI pending), unchanged, or needs you (and why); every branch it force-pushed; then what's ready to merge now: every PR whose status line says Ready (independent PRs included) and whose CI is green, plus the ones just pushed, as pending CI.
