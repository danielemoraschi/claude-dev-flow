---
name: quality-suite
description: Runs the full pre-merge quality chain on the current branch's changes end to end without stopping, so only commit and merge are left - in order dev-flow:code-modularizer, /simplify, optional /security-review, /code-review high --fix, dev-flow:clean-code, an independent change-reviewer pass, then lint, typecheck and tests, with each review step repeated until it stops finding material issues. Use whenever the user says "run the full suite", "run the quality suite", "full review chain", "run all the checks before merge", "polish this branch", or asks to get a change ready to merge, even if they don't name the individual skills. Only a run of this skill counts: doing its steps yourself isn't running it, and don't say a change went through the quality suite without its COMPLETE report.
---

# Quality suite

Run a fixed chain of review skills over the changes on the current branch, fixing as you go, so the change is ready to merge. Each step works on what the previous step left behind, which is why the order matters: structure first (modularize), then cleanup (simplify), then security and bug hunting on the cleaned code (security review, code review), then a final readability pass (clean code), then an independent reviewer who didn't write the code looks for what the chain missed (verification), then lint, typecheck and tests to prove nothing broke (verify).

## Run it end to end, in one go

The user calls `/dev-flow:quality-suite` and walks away. When you hand control back, every step (0–7) has run and the only thing left for them is to commit and merge. Handing back early is the failure this skill exists to prevent: the user reads "done-ish", commits, merges, and the remaining review steps never happen.

- **Create one todo per step (0–7) at the start**, named for how it runs ("Step 2: Skill simplify", "Step 6: agent change-reviewer"; in lite mode, per step that runs), add one for each repeat pass you decide on, and tick them off as you go. Before you end your turn, look at the list: if anything is unticked and not deliberately skipped (with a reason), you're not done; keep going.
- **Don't end your turn between steps.** Each skill in the chain ends with its own summary or report. That summary is intermediate: note what it changed, then immediately invoke the next step in the same turn. Don't show it to the user as a stopping point.
- **Don't ask permission between steps or for fixes.** Applying the fixes is the point of the run. Where a sub-skill would normally ask before applying a change, apply the ones that are clearly right and list any you deliberately left out in the final report.
- **Fix what breaks along the way.** If a step's fixes fail lint, typecheck or tests, fix them before moving on. Side issues you notice (a lint rule, a missing test for a fix you made) are part of the run, not reasons to stop.
- **Ask when the decision is genuinely the user's.** Some findings can't be settled from the code, the project docs or a sensible default: a fix that would change public API or behavior the project has frozen, two valid designs with real trade-offs, unclear intent behind the code, or a failure you can't diagnose. Ask then (with the AskUserQuestion tool if available, giving your recommendation), naming the step you're on and the steps still to run. When they answer, apply it and carry on to the end of the chain in the same turn. Don't ask about things with an obvious right answer; fix those.

If the user interrupts mid-run with something else, handle it, then resume at the step you were on. If they ask to commit, merge, push or delete the branch while steps are still pending, list the pending steps and ask whether to finish first before doing it, so a merge never slips through half-reviewed.

## What running a step means

A step counts only when it ran the way it's defined; doing its work yourself in its spirit isn't running it.

- **Steps 1 and 5** (`dev-flow:code-modularizer`, `dev-flow:clean-code`): a Skill tool call to that skill, then applying what it calls for. The call loads the skill's guidance; the evidence is the call plus what it changed.
- **Steps 2, 3 and 4** (`simplify`, `security-review`, `code-review`): a Skill tool call, made by you (or by the teammate running this suite), on every pass, with the scope as its argument (the changed files, and the worktree path when you work in one). These skills start their own review agents, which a subagent can't do, so never hand them to a subagent; their agents already review with fresh eyes.
- **Step 6**: the change-reviewer agent, as described there. **Step 7**: actually running the commands.

Reading the diff and fixing things yourself instead of a step is not running it. Skips come in two kinds:

- **Rule-based skips**, which the steps allow: step 1 in lite with no oversized file, step 3 with no security surface, step 6 for a trivial diff, in lite or with "no verification", and step 7 for a command the project doesn't define (name which). They keep the run COMPLETE.
- **Couldn't-run skips**: a step not run as defined, because the skill isn't available, it errored, a command exists but wasn't run, or its work was done some other way. For steps 2, 4, 5 and 7, and step 3 when the diff calls for it, the run is INCOMPLETE ("INCOMPLETE: step 2 simplify not available"), and the user decides whether to merge anyway. For steps 1 and 6, list it as "skipped (couldn't run)" with the reason; the run can still be COMPLETE.

`simplify`, `code-review` and `security-review` are built into Claude Code; `code-modularizer` and `clean-code` ship with this plugin. Use the built-in `code-review` (the one that takes `high --fix`), not a plugin's like `code-review:code-review`, which reviews pull requests differently.

## Lite mode

For small, contained changes there's a lighter run. It's lite when the request says so (it came from `/dev-flow:quality-suite-lite`, or the arguments say "lite"), or when the plan the user approved names lite as its quality gate. When you start this skill on your own (the user asked to get a change ready to merge), run the full chain: a lighter one is the user's call.

| Step | Full run | Lite |
|---|---|---|
| 0. Scope | as below | as below, plus a size warning |
| 1. code-modularizer | always | only if a changed file is over the code-modularizer limit (or CLAUDE.md's), measured with `wc -l` |
| 2. /simplify | repeats until it settles | one pass |
| 3. /security-review | when the diff calls for it | the same |
| 4. /code-review high --fix | repeats until it settles | one pass |
| 5. clean-code | repeats until it settles | one pass |
| 6. Independent verification | runs | skipped |
| 7. Verify | runs | the same |

In lite, "Repeat a step until it settles" doesn't apply: anything material a single pass leaves goes in the report. Everything else holds as in a full run: run to the end without stopping, don't ask permission between steps, fix what breaks, ask only for the user's decisions, follow the ground rules.

## Repeat a step until it settles

One pass of a step often isn't enough: its fixes open up new findings of the same kind, or a large diff is more than one pass covers. So after each of steps 1–5, decide whether that step needs another pass before moving on. How many passes each step gets depends on the change: one run might be one modularizer pass, two simplify, two code-review and two clean-code; another might be one, one, three and one.

- **Run the step again** (same skill, same scope, on the now-current code) when the pass applied many fixes (roughly 5 or more) or changed a meaningful amount of code, since fixes create new opportunities and can introduce new bugs; or when the diff is large (roughly 500+ changed lines or 15+ files, measured now as in step 0) and one pass is unlikely to have covered it.
- **Move on** when a pass finds nothing material: no findings, only nits, or only findings you already declined. A small diff with few or no findings needs only one pass.
- **Stop at 3 passes per step.** If the third pass still finds material issues, list them in the report rather than looping. If passes start undoing each other's changes, stop that step and report it.
- **Keep a list of findings you declined** (and why) and don't act on them again in later passes or later steps.
- **Only go forward.** A repeat pass happens right after the pass before it; later steps don't send you back to earlier ones. Steps 6 and 7 catch what they miss or break.

When you decide on another pass, add a todo for it, so the list always shows what's left.

## Use subagents where they help

Hand work to subagents when it keeps the run sharper or the main context lean; keep the decisions (whether to repeat, what to decline, when to ask the user) in the main conversation.

- **Repeat passes of steps 1 and 5**: run the step's skill in a fresh subagent, told to make the Skill tool call itself, given the scope and the declined list, so it reviews with fresh eyes instead of re-finding what the previous pass already settled. Steps 2–4 always run in your own thread (see "What running a step means").
- **Large diffs, steps 1 and 5**: split the step across subagents by disjoint groups of files, each owning its own files and making its own Skill call; merge their reports before deciding whether to repeat.
- **Never in parallel across steps or on the same files.** Steps run in order on what the previous one left, and two agents must never edit the same file.

Ask each subagent for what it changed, what it found but left, and anything that needs the user's decision.

## 0. Establish the scope

Work out which diff the suite covers before running anything:

- If the user names a scope (a commit, a commit range, or "what landed in X"), use that. This also covers running the chain after the fact on work that already merged: e.g. `git diff <sha>^ <sha>`, and land fixes as a follow-up on a new short-lived branch.
- Otherwise find the base: the branch the project merges into (check the project's CLAUDE.md, e.g. `next`, otherwise `main`/`master`). Use `git merge-base HEAD <base>`.
- The scope is the committed branch diff plus uncommitted and staged changes, plus new untracked files (`git ls-files --others --exclude-standard`), which `git diff` doesn't show.
- Record the size: `git diff --stat <merge-base>` (files changed, lines added/removed), adding the untracked files to the count.
- Note the plan or task the change implements, if the caller gives one or this conversation has one (its path or its text): step 6 checks the code against it.
- Keep every step on this scope. Some skills review only the uncommitted diff by default, so when the scope includes commits, pass it to each step explicitly (the base branch or commit range, or the changed paths, in whatever form that skill's arguments take).

In lite mode, check the diff is still small (roughly up to 100 changed lines, 5 files, and no new source modules). If it isn't:

- lite that an approved plan chose by size: the size was an estimate and it was wrong, so run the full suite instead, and say why;
- lite the user chose (asked for directly, or a plan's gate marked as the user's choice; a lite that arrives as an argument from a plan's gate follows the plan's marker, "chosen by size" or "the user's choice"): say so and recommend the full suite, then carry on in lite, since the user chose it knowing the change.

If the diff is empty, say so and stop. If the diff is documentation-only and the project's CLAUDE.md says doc-only changes skip the chain, say so and stop.

## 1. code-modularizer

Invoke the `dev-flow:code-modularizer` skill with the Skill tool on the changed files (in lite mode, only when a changed file is over the size limit; see Lite mode). Splitting oversized files first means later passes review the final layout instead of code that is about to move. Respect the project's own size limits and layering rules if CLAUDE.md states them.

## 2. /simplify

Invoke the `simplify` skill with the Skill tool on the branch diff. It fixes reuse, simplification and efficiency issues; it does not hunt for bugs.

## 3. /security-review (only when the changes call for it)

Decide from what the diff touches. Run `security-review` with the Skill tool when the changes involve any of:

- authentication, authorization, sessions, tokens, secrets or credentials
- parsing or rendering untrusted input (HTML/DOM injection, `innerHTML`, templates, URLs, deserialization)
- shell commands, file paths, SQL or other query building
- network requests, CORS, CSP, cookies, webhooks
- new or upgraded dependencies, build or CI configuration

Skip it for pure refactors, internal logic, tests, styling and docs, and say in the report that you skipped it and why.

## 4. /code-review high --fix

Invoke `code-review` with the Skill tool, args `high --fix`, then repeat as above. On a large diff, run at least a second pass even if the first was quiet; one high-effort pass rarely covers that much code.

In a git worktree (run the check where the scope lives, `git -C <that directory> rev-parse --git-dir`, and compare with `--git-common-dir`), still invoke it with the Skill tool, with args `high` only, without `--fix`: `--fix` applies its fixes outside the worktree. Pass the scope explicitly, apply the findings by hand in the worktree, and repeat as above.

## 5. clean-code

Invoke the `dev-flow:clean-code` skill with the Skill tool on the branch diff as the final readability pass (naming, function size, comments, error handling).

## 6. Independent verification

Skipped in lite mode.

The steps so far ran in your own stream: you fixed what they found and decided what to decline. Now have someone who didn't write the code look for what the chain missed, and sign off.

1. Skip this for a trivial diff (one file, under about 20 changed lines), or when the request says "no verification"; say which in the report.
2. Start the `dev-flow:change-reviewer` agent as a subagent, without a `name` (with agent teams enabled, a named subagent launches as a teammate). Its brief: the scope (base, range or changed paths, and whether uncommitted changes are included), the plan or task from step 0, the skills directory `${CLAUDE_SKILL_DIR}/..`, and a summary of this run so far: passes per step and every finding declined, with why. If that agent type isn't available in this session (a plugin update only loads agents after a restart), start an unnamed general-purpose subagent instead, told to read `${CLAUDE_SKILL_DIR}/../../agents/change-reviewer.md` and act as that agent, with the same brief. If neither is possible, say verification was skipped and why.
3. Fix each blocking and should-fix finding in place, or dispute it with a reason grounded in the code. Fix the nits that are clearly right.
4. If any blocking or should-fix finding was fixed or disputed, start a fresh reviewer the same way, adding round 1's findings and your response to each. Two rounds at most. If the user asks for another review pass, start a fresh change-reviewer whose brief says the user asked for it, with the earlier rounds' findings; it's extra to the two rounds. Handle its findings as in items 3 and 5 (a blocking finding you dispute goes to the user), then rerun step 7 and update the report if anything changed.
5. A blocking finding still held after the last round makes the run INCOMPLETE. Should-fix findings still held go in the report, for the user to decide.

## 7. Verify

Run the project's lint, typecheck and test commands (from CLAUDE.md or `package.json`/equivalent) so the fixes from the chain are proven not to break anything. Run the full suite after the last code change, not just the tests near it; if you change anything after verifying (a lint fix, a new test), verify again. If something fails, fix it and rerun; report anything you could not fix with its output.

## Ground rules

- If a skill in the chain can't be invoked (not available, or it errors), skip it and say so in the report; don't substitute a different one, or your own reading, for it. For steps 2, 4, 5 and 7 that's a couldn't-run skip, and the run is INCOMPLETE (see "What running a step means").
- Don't commit, push or merge; that's the user's part, after the run.
- Follow the project's CLAUDE.md where it adds to or narrows these steps.

## Report

Only once every step has run (or been deliberately skipped), end with a short summary. Its first line is the status: "COMPLETE, ready to commit and merge", or "INCOMPLETE" with the pending steps and the blocker; in lite mode, "COMPLETE (lite)" or "INCOMPLETE (lite)". It's COMPLETE only if steps 2, 4, 5 and 7 each have a line saying how they ran or a rule-based skip, step 3 ran or had a rule-based skip, and steps 1 and 6 ran or have a skip line with its reason.

Say a change went through the quality suite only when this report says COMPLETE.

- scope: base branch, files and lines changed
- per step: how it ran ("Skill: simplify, 2 passes", "Skill: clean-code, 3 fixes", "agent: change-reviewer, 1 round", "ran: npm test") or the skip and its kind ("skipped (rule): no security surface", "skipped (couldn't run): code-review errored"), why it stopped (settled, or hit the cap), and what it changed. A step with no such line didn't run
- findings left unfixed or declined, with the reason, especially from code-review; in lite mode, a material leftover from a single pass is listed with "consider the full suite"
- verification: rounds, verdict, findings by grade, what was fixed, and what's still held and why
- verify: commands run and their result
