---
name: clean-code
description: Readability and maintainability guidance in the spirit of Robert C. Martin's "Clean Code" - naming, function size and focus, comments, error handling, class and module design, tests. Use when writing, reviewing or refactoring code so it reads clearly to the next person, when designing an implementation plan, or when asked to "clean up", "make this readable", or review code for quality rather than bugs.
---

# Clean code

The goal is code another developer can read, trust and change without asking the author. Apply this to the code in scope (a diff, a file, a plan), and where the project has its own conventions (CLAUDE.md, a style guide, a linter config), those win over the generic advice here.

When reviewing, fix what's clearly better and leave the rest; don't churn code just to match a rule. Each change should make the code easier to read, not merely different.

## Names

- A name says why the thing exists and how it's used: `retryDelayMs`, not `d` or `val2`.
- Don't mislead: no `userList` for a set, no `isReady` that also starts a connection.
- Distinguish for real: `order` vs `orderSummary` tells a reader something; `orderData` vs `orderInfo` doesn't.
- Types and classes are nouns (`Invoice`, `RouteMatcher`); functions are verbs or questions (`sendInvoice`, `hasExpired`). Avoid filler like `Manager`, `Helper`, `Util` when a precise word exists.
- Prefer names you can say aloud and search for; save short names for tiny scopes.

## Functions

- Small, and doing one thing. If you can describe it with "and", it's probably two functions.
- One level of abstraction per function: a function that orchestrates steps shouldn't also be fiddling with string offsets.
- Few arguments. Zero to two is comfortable; three or more suggests a missing object or a function doing too much. Boolean flag arguments usually mean two functions.
- No hidden side effects: a function named `getX` shouldn't write anything.
- Read top-down: callers above callees, so the file reads like an outline that gets more detailed as you go.

## Comments

- Make the code say it first. A comment explaining a condition is often a function name waiting to happen (`if (isEligibleForRefund(order))`).
- Keep comments that code can't express: why a non-obvious choice was made, a link to the spec or bug, a warning about consequences, the intent behind a tricky regex.
- Remove comments that restate the code, are out of date, or mark positions; commented-out code belongs in version control, not the file.

## Formatting

- Related lines sit together; unrelated blocks are separated.
- Declare variables close to where they're used.
- High-level concepts near the top of a file, details further down.
- Follow the project's formatter; don't hand-format against it.

## Objects, data and modules

- Hide internals behind a small interface; callers shouldn't depend on how something is stored.
- Don't reach through objects (`a.b().c().d()`); ask the nearest collaborator for what you need.
- Keep plain data structures plain, and behavior-rich objects encapsulated; avoid hybrids that are half of each.
- One responsibility per class or module: one reason for it to change. When a file holds several concepts, split it (see the code-modularizer skill in this plugin).

## Errors

- Handle errors in one clear place rather than threading status codes through every caller; follow the language's and project's idiom (exceptions, `Result` types, error returns).
- Give errors context: what was attempted and why it failed.
- Don't return or pass `null`/`undefined` where an empty value, an optional type, or an error says it better, unless that's the project's documented convention.
- Never swallow an error silently.

## Tests

- Tests are code too: readable, one concept per test, named for the behavior they check.
- Fast, independent of each other, repeatable anywhere, pass/fail without manual inspection, and written alongside the code (ideally first, failing, then made to pass).

## Smells to watch for

Duplication; long functions or files; long argument lists; code that's hard to change in one place without breaking another; dead code; vague names; needless cleverness or speculative generality.

## Quick checklist

- Does each function do one thing, at one level of abstraction?
- Would a newcomer understand every name without reading the implementation?
- Could any comment be replaced by a better name or a small function?
- Are errors handled once, with context, and never swallowed?
- Is there a test that would fail if this change were reverted?
