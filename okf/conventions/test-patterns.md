---
type: Convention
title: Test patterns — layers, in-process MCP, spawned-bin crash injection, and virtual filesystems
description: "Seven canonical shapes for testing this family: Effect test-layer composition, the in-process MCP harness and direct caller, spawned-bin crash injection, duck-typed Vitest fixtures, process-level coordination, pure-function tests, and a virtual filesystem instead of a real temp tree."
tags: [testing, effect]
status: stable
stale_after: 2027-03-13T00:00:00Z
sources:
  - id: engine-testing
    resource: ../../packages/engine/src/testing/layers.ts
    title: makeTestLayer and DataStoreTestLayer
  - id: mcp-harness
    resource: ../../packages/mcp/__test__/utils/harness.ts
    title: The in-process MCP harness over Stdio.layerTest
  - id: mcp-caller
    resource: ../../packages/mcp/__test__/utils/caller.ts
    title: makeCaller — the direct handler-level caller
  - id: mcp-process
    resource: ../../packages/mcp/__test__/utils/mcp-process.ts
    title: spawnMcp and the crash-injection helper
  - id: build-report
    resource: ../../packages/sdk/src/utils/build-report.ts
    title: Duck-typed VitestTestModule / VitestTestCase interfaces
  - id: ensure-migrated
    resource: ../../packages/engine/src/utils/ensure-migrated.ts
    title: ensureMigrated's globalThis migration-promise cache
  - id: boundaries-scanner
    resource: ../../packages/sdk/__test__/utils/boundaries.ts
    title: Shared comment-stripping boundary scanner
  - id: workspace-layering-test
    resource: ../../packages/plugin/__test__/workspace-layering.test.ts
    title: The rank / DAG layering guardrail
  - id: memfs-walker
    resource: ../../packages/plugin/__test__/utils/memfs-walker.ts
    title: withMemfsWalker — the WalkerFileSystem adapter over a memfs volume
  - id: vitest-loader
    resource: ../../packages/mcp/__test__/resolve-vitest-node-entry.test.ts
    title: vitestLoader — a mutable holder for an unmockable dynamic import
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: cce9e25b133873713acdd58f0cfe2968d78daf3cadeb185770c8d777fc208fef
---

# Test patterns — layers, in-process MCP, spawned-bin crash injection, and virtual filesystems

## Pattern 1 — Compose Effect test layers instead of mocking a service's methods

Give each Effect service under test a `Test` layer that accumulates writes
into (or reads canned data from) a plain mutable container, and merge those
layers with `Layer.mergeAll` in place of the corresponding `Live` layer. The
migrated in-memory SQLite stack and five seeded presets ship from
`@vitest-agent/engine/testing`: `makeTestLayer(":memory:")` builds
`DataStoreLive` and `DataReaderLive` over a real `:memory:` `SqliteLayer` plus
its `MigratorLayer`[^engine-testing], and `DataStoreTestLayer` is the
pre-built `":memory:"` instance every unit test can reuse directly. Reach for
`makeTestLayer` (or the preset factories built on it) rather than hand-rolling
a fresh `SqliteLayer` / `MigratorLayer` pair in a test file.

## Pattern 2 — Drive the real MCP server layer over `Stdio.layerTest`, never a spawned process, for wire-level assertions

`packages/mcp/__test__/utils/harness.ts` builds the actual `ServerLayer` over
`Stdio.layerTest` queues and speaks JSON-RPC to it: `initialize`, `listTools`,
`callTool`, `sendRequest` (for `prompts/list` / `prompts/get`),
`sendNotification`, `seed` (populate the in-memory store the server reads),
and `session` (pin a fixture `McpSession`)[^mcp-harness]. Because the harness
serves the one real schema a wire client gets, a test cannot pass while
exercising a served-schema bug (a field the handler accepts that the schema
never declared) — there is no second schema to drift from the first. Provide
the test `Stdio` innermost in the layer merge: `DataStoreTestLayer` carries
`NodeServices.layer`, whose real process `Stdio` would otherwise win the
merge and leave the server listening on the Vitest worker's own stdin.

Use `packages/mcp/__test__/utils/caller.ts`'s `makeCaller(runtime, session?)`
instead when the assertion is about a tool handler's own logic: it decodes
params through the tool's `parameters` schema and invokes `toolHandlers[name]`
directly, giving full result-type narrowing with no wire
encoding[^mcp-caller]. Reach for the harness when the served schema, the
strict registrar, the dual channel, or a notification frame is the subject
under test; reach for the caller when the handler's own logic is.

## Pattern 2b — Spawn the built bin to prove a crash guard survives, and name that file `.e2e.test.ts`

A process-level `unhandledRejection` / `uncaughtException` guard cannot be
exercised in-process: a real uncaught throw inside the Vitest worker is not
something a test can safely trigger, and stubbing `process.on` proves only
that a handler was registered, never that the process survives it. Spawn the
**built** bin as a real child instead (`packages/mcp/__test__/utils/mcp-process.ts`'s
`spawnMcp`, `makeScratchProject`, and `handshake`, with `XDG_DATA_HOME`
pointed at a scratch directory), drive it over raw JSON-RPC on stdio, and
trigger the crash through an env-gated, fires-once injection hook
(`VITEST_AGENT_MCP_TEST_INJECT_CRASH`) scheduled for the event-loop turn
right after the transport connects, so ordering stays
deterministic[^mcp-process]. Assert on the thing that matters: the injected
error kind appears on stderr before a subsequent `ping` is sent, and `ping`
still answers over the same transport. The hook must stay env-gated so it can
never fire in a normal install, and the file must carry the `.e2e.test.ts`
suffix — a plain `.test.ts` classifies as `unit` and gets a 5 s timeout, which
a real spawn plus handshake routinely exceeds.

## Pattern 3 — Construct duck-typed Vitest fixtures instead of mocking the Vitest runtime

`buildAgentReport()` and reporter integration tests take plain object
literals shaped like `VitestTestModule` / `VitestTestCase` — interfaces
defined in `packages/sdk/src/utils/build-report.ts`[^build-report] — rather
than mocking Vitest's own runtime classes. A fixture module needs only the
methods the code under test actually calls (`state()`, `diagnostic()`,
`errors()`, `children.allTests()`, `project.name`), so a test stays legible
as data instead of as a mock-configuration script.

## Pattern 4 — Reset the `globalThis` migration cache between process-coordination test cases

`ensureMigrated` shares one in-flight promise per `dbPath` across an entire
process, cached on `globalThis` behind `Symbol.for("vitest-agent/migration-promises")`[^ensure-migrated].
Its own test-only export, `_resetMigrationCacheForTesting`, clears that cache
between cases so each test starts from a clean slate rather than reusing
another test's already-resolved promise for the same `dbPath`. Cover at
least: a fresh DB migrating without error, concurrent calls against the same
`dbPath` sharing one promise, distinct `dbPath`s yielding independent
promises, and several concurrent callers serializing without a `SQLITE_BUSY`
error.

## Pattern 4b — Guard the rank rule and package boundaries with structural scans, not spot checks

Two structural suites run as ordinary unit tests and guard the family's
layering contract for every package, not just the one under active edit.
Every `packages/{sdk,engine,cli,mcp}/__test__/boundaries.test.ts` runs over a
shared comment-stripping scanner (`__test__/utils/boundaries.ts`: `walkTs`,
`referencesProcess`, `importSpecifiers`), built regex-literal aware because an
unguarded scanner once let a regex containing `/*` swallow the rest of a
file[^boundaries-scanner]. `packages/plugin/__test__/workspace-layering.test.ts`
reads every workspace manifest and asserts that every dependency edge points
to a strictly lower rank, that `cli` and `mcp` never depend on each other,
and that the graph's topological sort consumes every node[^workspace-layering-test] —
adding a workspace package means adding it to `LAYER_RANKS` in that suite's
`utils/workspace-graph.ts`, not merely wiring its `package.json`.

## Pattern 5 — Test CLI and engine-program logic as plain pure functions, not through the command tree

CLI commands are thin wrappers over `effect/unstable/cli` `Command`
definitions and are not exercised directly. The testable logic — formatting,
program bodies — lives in plain functions (`packages/cli/src/lib/format-*.ts`,
`packages/engine/src/programs/*.ts`) that take a domain input
(`AgentReport`, `CoverageReport`, an `env`/`cwd` pair) and return a rendered
string or a plain result, so the test calls the function directly instead of
spawning or stubbing the command tree around it.

## Pattern 6 — Seed a virtual filesystem instead of `mkdtemp`-ing a real temp tree

Filesystem-touching tests mount an `@effected/memfs` volume rather than
creating a real temporary directory, and the swap takes one of two shapes.
Where the code under test already reads through Effect's `FileSystem`
service, the swap is a layer swap: provide `MemoryFileSystem.layerWith(seed)`
(plus `Path.layer`) in place of `NodeServices.layer` /
`NodeFileSystem.layer`, and drop the `writeFileSync` / `rmSync` scaffolding
entirely. Where the code takes the injected `WalkerFileSystem` port instead
(`@vitest-agent/plugin`'s discovery walkers), tests pass an adapter over a
memfs volume — `packages/plugin/__test__/utils/memfs-walker.ts`'s
`withMemfsWalker`[^memfs-walker] — built on the volume's literal *inspection*
view rather than its link-resolving `syncFileSystem` port, because the
walkers must not follow symlinks and the inspection view is honest about
them. That same adapter answers `mtimeMs` from the volume's own `mtime`,
which is what makes a discovery cache's signature-invalidation path testable
without touching a real disk. Seeding a volume also makes the awkward cases —
symlink loops, an unreadable directory, an exact modification time no
`utimes` call would reliably produce — cheap and deterministic instead of
flaky.

## Pattern 7 — Reach for a mutable loader object when `vi.mock` cannot reach the import

Some imports are unmockable by construction: `vitest/node` is imported
through a computed specifier so a tool can drive the physical Vitest copy
installed at the project under test, and Vitest's own vite-node externalizes
the vitest package for every importer, special-casing only AST-literal
`import("vitest/node")` call sites for `vi.mock` interception — a
`vi.mock("vitest/node", ...)` against the computed form silently no-ops while
the real module loads, producing a test that passes while exercising
nothing. The pattern is a plain exported object whose `load` property holds
the dynamic import (`vitestLoader`); a test assigns over `vitestLoader.load`
and restores it afterward, asserting on the specifier the tool
computed[^vitest-loader]. When a module boundary sits outside the mocking
framework's reach, an exported mutable holder is the seam to reach for —
never a deeper mock aimed at the same unreachable boundary.

## Related concepts

- [test-layout convention](./test-layout.md) states the file-placement and
  `.e2e.test.ts`-naming rule these patterns assume.
- [ranked-layering invariant](../invariants/ranked-layering.md) and
  [package-boundaries invariant](../invariants/package-boundaries.md) are the
  properties Pattern 4b's two suites enforce.
- [coverage-targets convention](./coverage-targets.md) is the threshold policy
  these patterns are written against.

[^engine-testing]: ../../packages/engine/src/testing/layers.ts
[^mcp-harness]: ../../packages/mcp/**test**/utils/harness.ts
[^mcp-caller]: ../../packages/mcp/**test**/utils/caller.ts
[^mcp-process]: ../../packages/mcp/**test**/utils/mcp-process.ts
[^build-report]: ../../packages/sdk/src/utils/build-report.ts
[^ensure-migrated]: ../../packages/engine/src/utils/ensure-migrated.ts
[^boundaries-scanner]: ../../packages/sdk/**test**/utils/boundaries.ts
[^workspace-layering-test]: ../../packages/plugin/**test**/workspace-layering.test.ts
[^memfs-walker]: ../../packages/plugin/**test**/utils/memfs-walker.ts
[^vitest-loader]: ../../packages/mcp/**test**/resolve-vitest-node-entry.test.ts
