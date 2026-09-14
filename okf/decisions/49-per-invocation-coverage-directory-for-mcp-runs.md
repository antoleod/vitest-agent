---
type: Decision
status: draft
title: Per-Invocation Coverage Directory for MCP Runs
description: Every run_tests call gets its own throwaway coverage reports directory so concurrent runs never clobber each other's on-disk artifacts.
tags: [mcp, testing]
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: 3e29ed0178032d74221edc73b51470714a16b6d3d93608cebfe43761329fac32
---

# Per-Invocation Coverage Directory for MCP Runs

## Context

Vitest's v8 coverage provider removes its reports directory at run start
(`coverage.clean` defaults to `true`). Two runs sharing one checkout — an
MCP `run_tests` call alongside a Bash `vitest run`, or two concurrent MCP
calls — share `./coverage` and delete each other's `.tmp` files mid-flight,
so one dies with an `ENOENT` on a `coverage-N.json` file
(`packages/mcp/src/tools/run-tests.ts:186-188`). Forcing
`coverage.enabled: false` for MCP runs was tried and reverted because it
overrides intentional user configuration, and serializing MCP runs against
every other Vitest process in the checkout is not enforceable.

## Decision

`makeCoverageDirOverride()`
(`packages/mcp/src/tools/run-tests.ts:199-202`) gives each `run_tests`
invocation its own `mkdtemp`-created `coverage.reportsDirectory`. The
result is spread onto the `createVitest` overrides as a field-level merge
(`packages/mcp/src/tools/run-tests.ts:964`, `coverage: covOverride.coverage`)
so `enabled`, the provider, and thresholds all still come from the user's
own config — only `reportsDirectory` is replaced.

The override is created *inside* the tool's `try` block
(`packages/mcp/src/tools/run-tests.ts:945-951`), not before it, so a
throwing `mkdtempSync` — a full or read-only tmpdir — is caught by the
surrounding catch and returns the tool's normal `{ kind: "error", message
}` envelope instead of propagating raw out of the handler. Cleanup is a
best-effort `rmSync` in a nested `finally`
(`packages/mcp/src/tools/run-tests.ts:1149-1160`) that runs even when
`vitest?.close()` itself rejects.

## Alternatives rejected

- **Forcing `coverage.enabled: false` on every MCP run:** already tried and
  reverted — it silently discarded a user's intentional "coverage on by
  default" configuration and forced a parallel Bash `--coverage` call just
  to populate `file_coverage` rows.
- **Serializing MCP runs against every other Vitest process in the
  checkout:** rejected as unenforceable — there is no reliable way for the
  MCP process to detect or lock against an arbitrary concurrent `vitest
  run` invoked outside its control.

## Consequences

- Final coverage artifacts (HTML, LCOV) from MCP-driven runs land in the
  throwaway directory rather than `./coverage`, and are deleted afterward.
  This is acceptable because the MCP path never reads coverage from disk —
  the plugin's `CoverageAnalyzer` consumes the in-memory `CoverageMap` via
  `onCoverage` and persists it to SQLite, which is what every MCP coverage
  tool queries. A user who wants the on-disk report runs Vitest directly.
- This decision covers the MCP `run_tests` path only. The plain-CLI half
  of the same clobber — an agent's Bash `vitest run` racing another
  process in the checkout — is a separate, still-open surface: mirroring
  the per-process directory inside the plugin's own Vitest configuration
  for the `agent` executor would need its own decision.
- A caller relying on `./coverage` existing after an MCP-driven run will
  not find it there; any tooling that reads coverage artifacts off disk
  must be pointed at SQLite instead.
