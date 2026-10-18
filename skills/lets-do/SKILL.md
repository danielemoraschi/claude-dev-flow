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

Their content should shape the plan itself, not show up as a line saying "follow clean-code" or "apply code-modularizer". A plan that just names them hasn't applied them. Make it concrete:

- **Files and modules**: the plan names each new or changed file, the one concept it holds, and roughly how big it will get; where an existing file would grow past the modularizer's limits, the plan splits it as part of the work, not afterwards.
- **Functions and names**: the main functions and types are named intention-revealingly in the plan, each doing one thing, with boundaries drawn where clean-code would draw them.
- **Each implementation step** carries the specific clean-code and modularity decisions for that step (what goes where, what gets extracted, what stays small), so they're built in as the code is written.

Where the project's CLAUDE.md sets its own rules (file-size limits, layering, naming), those win over the skills' generic advice.

## Quality gate

Add a `/dev-flow:quality-suite` pass to the plan's quality gates (add a quality-gates section if the plan has none), to run on the finished change before it's committed and merged. This one is a gate to run, not guidance to embed.

## Overrides

The user's own words win. If the task says to drop or change any default (e.g. "ignore code-modularizer", "no quality-suite", "skip clean-code"), follow that instead.
