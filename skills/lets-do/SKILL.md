---
name: lets-do
description: Start planning a new piece of work with the user's standing defaults - the clean-code and code-modularizer skills read up front and applied throughout the plan, and a /quality-suite pass as a quality gate. Invoked as /dev-flow:lets-do followed by the task; anything the user says in the task overrides these defaults.
disable-model-invocation: true
---

# Let's do

The task: $ARGUMENTS

If no task is given, ask what to do before anything else.

## First: read the two skills, every time

Your first tool calls, before reading any project code, read these two files in full:

- `${CLAUDE_SKILL_DIR}/../clean-code/SKILL.md`
- `${CLAUDE_SKILL_DIR}/../code-modularizer/SKILL.md`

Do this on every run, even if you read them earlier in this conversation: earlier reads may have been compacted into a summary, and a summary isn't the guidance. Read them as guidance; don't invoke them as skills or run their workflows on the existing code.

Then say in one line that you read them, with each file's line count, so the user can see it happened. If a file can't be read, stop and say so instead of planning without it.

## Build the plan with them embedded

Their content should shape the plan itself, and the plan should show where it did. A line saying "follow clean-code" or "apply code-modularizer" applies nothing; a plan that's merely tidy doesn't show the skills were used either. Give the plan this shape, so the decisions are visible in it and a missing one shows:

- **Design decisions**, near the top: the clean-code and modularity decisions this plan makes, one per line, each with the rule it applies (from the skill, in a few words) and the step where it lands. For example:
  - *`options.ts` split out of `map.ts`, so `map.ts` stays composition only*: code-modularizer, one concept per file. Step 2.
  - *`checkAttachable` renamed `resolveAttachable`, since it now returns the element*: clean-code, a name says what it does. Step 1.
  - *Invalid options throw `invalid-input` at the boundary, not a `TypeError` deep inside*: clean-code, handle errors in one clear place, with context. Step 2.

  Cover every file the change creates or grows, the main new functions and types, and the error handling. A decision listed here with no step carrying it, or a step making a structural choice not listed here, means the plan isn't finished.
- **File map**, before the steps: every new or changed file, with the one concept it holds, its current size (measure existing files with `wc -l`, don't guess) and its expected size after the change. Any file that would pass the modularizer's limits gets split as a step of this plan, not afterwards.
- **Each implementation step** says, besides what it does:
  - *Structure*: which files it creates or changes, and what it extracts or keeps out of each one.
  - *Code*: the functions and types it adds or changes, named intention-revealingly, each doing one thing, with few arguments and no boolean flags or hidden side effects.
  - *Errors*: how the step's failure cases are handled and where, in the project's idiom.
  - *Tests*: the tests that prove the step, named for the behavior they check.

A step whose structure or code line is vague ("clean up as needed", "helpers", "a utils file") isn't planned yet. Where the project's CLAUDE.md sets its own rules (file-size limits, layering, naming), those win over the skills' generic advice.

## Quality gate

Add a `/dev-flow:quality-suite` pass to the plan's quality gates (add a quality-gates section if the plan has none), to run on the finished change before it's committed and merged. This one is a gate to run, not guidance to embed.

## Overrides

The user's own words win. If the task says to drop or change any default (e.g. "ignore code-modularizer", "no quality-suite", "skip clean-code"), follow that instead.
