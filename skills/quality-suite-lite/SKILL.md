---
name: quality-suite-lite
description: The quality-suite chain with small-change settings - one pass each of /simplify, /code-review high --fix and dev-flow:clean-code, /security-review when the diff calls for it, code-modularizer only for an oversized file, no independent reviewer, then lint, typecheck, tests and the project's own gates. For small, contained changes; anything added after the command (a scope, a base branch) is passed on.
disable-model-invocation: true
argument-hint: "[scope or notes, optional]"
---

# Quality suite (lite)

Invoke the `dev-flow:quality-suite` skill with the arguments `lite $ARGUMENTS`, and follow it in lite mode, end to end.

Don't run the steps yourself from memory: the chain's rules, including what lite changes, live in quality-suite.
