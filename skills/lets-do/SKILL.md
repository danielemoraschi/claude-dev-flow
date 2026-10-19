---
name: lets-do
description: Start planning a new piece of work with the user's standing defaults - the clean-code and code-modularizer skills read up front and applied throughout the plan, and a /quality-suite pass as a quality gate. Invoked as /dev-flow:lets-do followed by the task; anything the user says in the task overrides these defaults.
disable-model-invocation: true
---

# Let's do

The task: $ARGUMENTS

If no task is given, ask what to do before anything else.

## Before planning: read the two skills

Read these in full before you design anything. They ship in this plugin, next to this skill: from this skill's base directory, they are

- `../clean-code/SKILL.md`
- `../code-modularizer/SKILL.md`

Read the files directly, as guidance; don't invoke them as skills or run their workflows on the existing code.

## Build the plan with them embedded

Their content should shape the plan itself, not show up as a line saying "follow clean-code" or "apply code-modularizer". A plan that just names them hasn't applied them. Give the plan this shape, so the decisions are visible in it and a missing one shows:

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
