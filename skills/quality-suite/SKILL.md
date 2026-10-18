---
name: quality-suite
description: Runs the full pre-merge quality chain on the current branch's changes end to end without stopping, so only commit and merge are left - in order dev-flow:code-modularizer, /simplify, optional /security-review, /code-review high --fix, dev-flow:clean-code, then lint, typecheck and tests, with each step repeated until it stops finding material issues. Use whenever the user says "run the full suite", "run the quality suite", "full review chain", "run all the checks before merge", "polish this branch", or asks to get a change ready to merge, even if they don't name the individual skills.
---

# Quality suite

Run a fixed chain of review skills over the changes on the current branch, fixing as you go, so the change is ready to merge. Each step works on what the previous step left behind, which is why the order matters: structure first (modularize), then cleanup (simplify), then security and bug hunting on the cleaned code (security review, code review), then a final readability pass (clean code), then lint, typecheck and tests to prove nothing broke (verify).

## Run it end to end, in one go

The user calls `/dev-flow:quality-suite` and walks away. When you hand control back, every step (0–6) has run and the only thing left for them is to commit and merge. Handing back early is the failure this skill exists to prevent: the user reads "done-ish", commits, merges, and the remaining review steps never happen.

- **Create one todo per step (0–6) at the start**, add one for each repeat pass you decide on, and tick them off as you go. Before you end your turn, look at the list: if anything is unticked and not deliberately skipped (with a reason), you're not done; keep going.
- **Don't end your turn between steps.** Each skill in the chain ends with its own summary or report. That summary is intermediate: note what it changed, then immediately invoke the next step in the same turn. Don't show it to the user as a stopping point.
- **Don't ask permission between steps or for fixes.** Applying the fixes is the point of the run. Where a sub-skill would normally ask before applying a change, apply the ones that are clearly right and list any you deliberately left out in the final report.
- **Fix what breaks along the way.** If a step's fixes fail lint, typecheck or tests, fix them before moving on. Side issues you notice (a lint rule, a missing test for a fix you made) are part of the run, not reasons to stop.
- **Ask when the decision is genuinely the user's.** Some findings can't be settled from the code, the project docs or a sensible default: a fix that would change public API or behavior the project has frozen, two valid designs with real trade-offs, unclear intent behind the code, or a failure you can't diagnose. Ask then (with the AskUserQuestion tool if available, giving your recommendation), naming the step you're on and the steps still to run. When they answer, apply it and carry on to the end of the chain in the same turn. Don't ask about things with an obvious right answer; fix those.

If the user interrupts mid-run with something else, handle it, then resume at the step you were on. If they ask to commit, merge, push or delete the branch while steps are still pending, list the pending steps and ask whether to finish first before doing it, so a merge never slips through half-reviewed.

`simplify`, `code-review` and `security-review` are built into Claude Code; `code-modularizer` and `clean-code` ship with this plugin. Use the built-in `code-review` (the one that takes `high --fix`), not a plugin's like `code-review:code-review`, which reviews pull requests differently.

## Repeat a step until it settles

One pass of a step often isn't enough: its fixes open up new findings of the same kind, or a large diff is more than one pass covers. So after each of steps 1–5, decide whether that step needs another pass before moving on. How many passes each step gets depends on the change: one run might be one modularizer pass, two simplify, two code-review and two clean-code; another might be one, one, three and one.

- **Run the step again** (same skill, same scope, on the now-current code) when the pass applied many fixes (roughly 5 or more) or changed a meaningful amount of code, since fixes create new opportunities and can introduce new bugs; or when the diff is large (roughly 500+ changed lines or 15+ files, measured now as in step 0) and one pass is unlikely to have covered it.
- **Move on** when a pass finds nothing material: no findings, only nits, or only findings you already declined. A small diff with few or no findings needs only one pass.
- **Stop at 3 passes per step.** If the third pass still finds material issues, list them in the report rather than looping. If passes start undoing each other's changes, stop that step and report it.
- **Keep a list of findings you declined** (and why) and don't act on them again in later passes or later steps.
- **Only go forward.** A repeat pass happens right after the pass before it; later steps don't send you back to earlier ones. Step 6 catches anything they break.

When you decide on another pass, add a todo for it, so the list always shows what's left.

## Use subagents where they help

Hand work to subagents when it keeps the run sharper or the main context lean; keep the decisions (whether to repeat, what to decline, when to ask the user) in the main conversation.

- **Repeat passes**: run the step's skill in a fresh subagent, given the scope and the declined list, so it reviews with fresh eyes instead of re-finding what the previous pass already settled.
- **Large diffs**: split one step across subagents by disjoint groups of files, each owning its own files; merge their reports before deciding whether to repeat.
- **Never in parallel across steps or on the same files.** Steps run in order on what the previous one left, and two agents must never edit the same file.

Ask each subagent for what it changed, what it found but left, and anything that needs the user's decision.

## 0. Establish the scope

Work out which diff the suite covers before running anything:

- If the user names a scope (a commit, a commit range, or "what landed in X"), use that. This also covers running the chain after the fact on work that already merged: e.g. `git diff <sha>^ <sha>`, and land fixes as a follow-up on a new short-lived branch.
- Otherwise find the base: the branch the project merges into (check the project's CLAUDE.md, e.g. `next`, otherwise `main`/`master`). Use `git merge-base HEAD <base>`.
- The scope is the committed branch diff plus uncommitted and staged changes, plus new untracked files (`git ls-files --others --exclude-standard`), which `git diff` doesn't show.
- Record the size: `git diff --stat <merge-base>` (files changed, lines added/removed), adding the untracked files to the count.
- Keep every step on this scope. Some skills review only the uncommitted diff by default, so when the scope includes commits, pass it to each step explicitly (the base branch or commit range, or the changed paths, in whatever form that skill's arguments take).

If the diff is empty, say so and stop. If the diff is documentation-only and the project's CLAUDE.md says doc-only changes skip the chain, say so and stop.

## 1. code-modularizer

Invoke the `dev-flow:code-modularizer` skill on the changed files. Splitting oversized files first means later passes review the final layout instead of code that is about to move. Respect the project's own size limits and layering rules if CLAUDE.md states them.

## 2. /simplify

Invoke the `simplify` skill on the branch diff. It fixes reuse, simplification and efficiency issues; it does not hunt for bugs.

## 3. /security-review (only when the changes call for it)

Decide from what the diff touches. Run `security-review` when the changes involve any of:

- authentication, authorization, sessions, tokens, secrets or credentials
- parsing or rendering untrusted input (HTML/DOM injection, `innerHTML`, templates, URLs, deserialization)
- shell commands, file paths, SQL or other query building
- network requests, CORS, CSP, cookies, webhooks
- new or upgraded dependencies, build or CI configuration

Skip it for pure refactors, internal logic, tests, styling and docs, and say in the report that you skipped it and why.

## 4. /code-review high --fix

Invoke `code-review` with args `high --fix`, then repeat as above. On a large diff, run at least a second pass even if the first was quiet; one high-effort pass rarely covers that much code.

## 5. clean-code

Invoke the `dev-flow:clean-code` skill on the branch diff as the final readability pass (naming, function size, comments, error handling).

## 6. Verify

Run the project's lint, typecheck and test commands (from CLAUDE.md or `package.json`/equivalent) so the fixes from the chain are proven not to break anything. Run the full suite after the last code change, not just the tests near it; if you change anything after verifying (a lint fix, a new test), verify again. If something fails, fix it and rerun; report anything you could not fix with its output.

## Ground rules

- If a skill in the chain is not available, skip it and say so in the report; don't substitute a different one silently.
- Don't commit, push or merge; that's the user's part, after the run.
- Follow the project's CLAUDE.md where it adds to or narrows these steps.

## Report

Only once every step has run (or been deliberately skipped), end with a short summary. Its first line is the status: "COMPLETE, ready to commit and merge", or "INCOMPLETE" with the pending steps and the blocker.

- scope: base branch, files and lines changed
- per step: ran / skipped (why), number of passes and why it stopped (settled, or hit the cap), and what it changed
- findings left unfixed or declined, with the reason, especially from code-review
- verify: commands run and their result
