---
name: code-modularizer
description: Breaks large or multi-purpose code files into small, cohesive modules (target 100-200 lines, never over about 300), and designs new code that way from the start. Use when a file has grown too large or mixes several concerns, when reviewing recent changes for modular structure, when planning where new code should live, or when asked to split, restructure or modularize code - in any language (TypeScript/React, Python, Go, etc.).
---

# Code modularizer

Keep every file small and about one thing. Smaller files are easier to read, test, review and change, and they make the structure of the code visible from the file tree. The project's own limits and layering rules (CLAUDE.md, lint config, an import-layer check) win over the defaults here.

## Defaults

- Target 100-200 lines per file; split before a file passes about 300.
- One concept per file: a reader should be able to say what the file is for in one short phrase.
- When unsure, slightly more files beats slightly larger ones, but keep related code close (same directory) before scattering it.
- Split by concern, don't add layers: separate files for separate concerns, but no new wrappers, interfaces, base classes or options the code doesn't need.

## How to work

1. **Measure**: line counts of the files in scope, and the responsibilities each one holds.
2. **Find the seams**: distinct concerns in one file, code that changes for different reasons, pure helpers mixed with stateful or I/O code.
3. **Map dependencies**: decide what can move without creating circular imports or breaking the project's layering.
4. **Extract**, dependencies first: create the new files, move the code, update imports and exports, keep behavior identical.
5. **Verify**: tests, typecheck and lint still pass; each new module can be tested on its own.

When designing new code (e.g. in a plan), do the same up front: name each file, the one concept it holds and its expected size, so nothing has to be split later.

## Common extractions

These are typical; match the project's existing structure and naming over these.

- **Pure logic** (calculations, parsing, formatting) out of components, handlers or classes into its own module.
- **Stateful logic** into its own unit: a React custom hook (`useThing.ts`), a service class, a store.
- **Sub-components or sub-handlers** that render or act independently.
- **I/O boundaries**: data access, API clients and external services behind their own modules, keeping route handlers and UI thin.
- **Types, constants, error classes** into dedicated files when they're shared or sizable.
- **Tests** mirroring the new source layout.

## Applying changes

Apply extractions with clear boundaries directly. Ask only when the boundary is genuinely ambiguous (two reasonable splits with real trade-offs, or a move that would change a public API); explain briefly why each extraction helps, not just what moved. Preserve the existing naming conventions and code style.

## Checks after a split

- No file over the limit; each has a single clear purpose.
- No circular dependencies; imports respect the project's layers.
- Public exports unchanged unless the change intended otherwise.
- Existing tests pass; new modules are independently testable.
