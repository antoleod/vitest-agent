---
type: Invariant
title: Package boundaries — process reads and forbidden imports
description: Each of sdk, engine, cli, and mcp carries a boundaries.test.ts that scans its own src/ for forbidden process reads and forbidden cross-package imports, pinning the platform-free core, the process-free engine, and the two front ends' narrow process allowlists.
tags: [architecture, effect]
resource: ../../packages/sdk/__test__/boundaries.test.ts
sources:
  - id: sdk-boundaries
    resource: ../../packages/sdk/__test__/boundaries.test.ts
  - id: engine-boundaries
    resource: ../../packages/engine/__test__/boundaries.test.ts
  - id: cli-boundaries
    resource: ../../packages/cli/__test__/boundaries.test.ts
  - id: mcp-boundaries
    resource: ../../packages/mcp/__test__/boundaries.test.ts
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: 0b55fc8f48bee2dc240f1902e0c4f8ecfe416a53e39b3d8ddebf564c341e270f
---

# Package boundaries — process reads and forbidden imports

## Property

Four packages — `@vitest-agent/sdk`, `@vitest-agent/engine`,
`@vitest-agent/cli`, and `@vitest-agent/mcp` — each carry a source-scanning
test, `__test__/boundaries.test.ts`, that pins two properties about every
file under that package's own `src/`: which files, if any, may reference
the global `process` object, and which sibling packages, if any, that
package's source may import. Every one of the four tests shares the same
comment-stripping scanner (`walkTs`, `referencesProcess`,
`importSpecifiers`, `VERSION_TOKEN`), so the four sets of assertions below
are variations on one mechanism, not four independent implementations.

- **sdk** — no file under `src/` may import `node:*`, `@effect/platform-node`,
  `@effect/sql-sqlite-node`, or any `@effected/*` package, and no file may
  reference `process.` anywhere[^sdk-boundaries].
- **engine** — no file under `src/` may reference `process.` anywhere,
  with no allowlist at all, and no file may import
  `@vitest-agent/cli`, `@vitest-agent/mcp`, `@vitest-agent/plugin`,
  `@vitest-agent/reporter`, or `@vitest-agent/ui`[^engine-boundaries].
- **cli** — `process` may be referenced only in `bin.ts`, `main.ts`,
  `version.ts`, or a file under `commands/**`; no file under `src/` may
  import `@vitest-agent/mcp`, `@vitest-agent/plugin`,
  `@vitest-agent/reporter`, or `@vitest-agent/ui`[^cli-boundaries].
- **mcp** — `process` may be referenced only in `bin.ts`, `main.ts`,
  `version.ts`, or `tools/run-tests.ts`; no file under `src/` may import
  `@vitest-agent/cli`, `@vitest-agent/plugin`, `@vitest-agent/reporter`,
  `@vitest-agent/ui`, `@modelcontextprotocol/sdk`, `@trpc/server`, or
  `zod`[^mcp-boundaries].

Across all four, the literal token `process.env.__PACKAGE_VERSION__` is
the one universally exempt reference, and it may appear only in that
package's own `version.ts` — every test asserts the file list that
contains the token equals exactly `["version.ts"]`, independent of the
package's general process-reference rule.

## Mechanism

Each `boundaries.test.ts` walks every `.ts` file under its own package's
`src/` (`walkTs`), reads it, strips comments, and runs two checks per
file: `referencesProcess(source)` (a textual scan for `process.`
occurrences) and `importSpecifiers(source)` (every static import
specifier the file declares). cli and mcp additionally allowlist a fixed
set of relative paths — checked by exact match on the path relative to
`src/`, or by a `startsWith` prefix for `commands/`[^cli-boundaries] — and
only fail a file for a `process` reference when that file's relative path
is not on the allowlist[^mcp-boundaries]. sdk and engine carry no
allowlist at all: `referencesProcess` failing on *any* file, anywhere
under `src/`, fails the test[^sdk-boundaries][^engine-boundaries]. Every
one of the four tests separately asserts the token-user file list equals
`["version.ts"]`, so a stray `process.env.__PACKAGE_VERSION__` reference
outside `version.ts` fails even on a file that is otherwise allowed to
read `process` for other reasons (cli's `commands/**`, mcp's
`tools/run-tests.ts`).

Because the scanner strips comments before matching, a reference inside a
code comment or a docstring cannot produce a false positive; because it
matches on the literal substring `process.`, a local variable or
destructured binding named `process` would still false-positive, though
no source file in the family does this.

## What a refactor would have to break

Because these are textual source scans over every file in `src/`, not a
lint rule scoped to an entry point, the properties hold for a file the
moment it is added to the tree — there is no opt-out short of editing the
allowlist array in the test itself. A refactor that moved a
platform-bound helper (SQLite, `@effected/xdg`, `std-env`'s `isAgent`)
into sdk would fail `sdk-boundaries` on the forbidden-import check the
moment the import statement lands, independent of whether that helper is
ever called. A refactor that added a `process.cwd()` read to an engine
service — for example, to shortcut passing `cwd` as an explicit
parameter — would fail `engine-boundaries` immediately, since engine's
rule carries no allowlist to hide behind; every ambient input engine
needs must arrive as a parameter from the front end that calls it. A
refactor that had `@vitest-agent/cli` import something from
`@vitest-agent/mcp` (or the reverse) to share a utility would fail the
corresponding forbidden-import check rather than the
[ranked-layering](./ranked-layering.md) test, since a same-rank edge
between the two front ends is not itself a rank violation — this is the
second, source-level half of that same prohibition. And moving a
`process.env` read for a new CLI subcommand into `lib/` instead of
`commands/`, or a new MCP tool's `process.env` mutation into a file other
than `tools/run-tests.ts`, would fail the corresponding allowlist check
even though the new code is otherwise correct.

The four tests do not check whether a *declared* dependency actually
gets used — see
[Invariant: Ranked layering](./ranked-layering.md) for the companion
manifest-level graph check the same #412 restructuring produced. Together
the two tests are what let [Decision 70](../decisions/70-carrier-pattern-and-ranked-layering.md)
and [Decision 71](../decisions/71-effect-native-mcp-server.md) claim the
platform-free core, the process-free engine, and the two front ends'
narrow entry contracts as properties of the tree rather than descriptions
of intent — see
[Convention: Front-end entry contract](../conventions/front-end-entry-contract.md)
for the `bin.ts` / `main.ts` / `index.ts` / `version.ts` split these
allowlists exist to protect.

[^sdk-boundaries]: `../../packages/sdk/__test__/boundaries.test.ts`
[^engine-boundaries]: `../../packages/engine/__test__/boundaries.test.ts`
[^cli-boundaries]: `../../packages/cli/__test__/boundaries.test.ts`
[^mcp-boundaries]: `../../packages/mcp/__test__/boundaries.test.ts`
