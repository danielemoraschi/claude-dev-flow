# dev-flow

A Claude Code plugin for planning and finishing changes.

- `/dev-flow:lets-do <task>`: manual-only (Claude won't load it on its own). Plans the task with clean-code and modularity built into the plan from the start, and a `/quality-suite` pass as a quality gate. Anything you say in the task overrides the defaults ("ignore code-modularizer", "no quality-suite").
- `/dev-flow:quality-suite`: runs the pre-merge chain end to end on the current branch, fixing as it goes: code-modularizer, `/simplify`, `/security-review` (when the change calls for it), `/code-review high --fix` (twice when warranted), clean-code, then lint, typecheck and tests. All that's left is to commit and merge.
- `clean-code` and `code-modularizer`: the guidance the other two use; also usable on their own.

`/simplify`, `/code-review` and `/security-review` are built into Claude Code.

The short forms `/lets-do` and `/quality-suite` usually work too, unless another skill has the same name.

## Install

From a clone or this folder:

```
/plugin marketplace add /path/to/dev-flow
/plugin install dev-flow@dev-flow
```

From GitHub:

```
/plugin marketplace add danielemoraschi/claude-dev-flow
/plugin install dev-flow@dev-flow
```

The repo is private: the installing account needs git access to it (`gh auth login` or a git credential helper), and `GITHUB_TOKEN` set for background auto-updates.
