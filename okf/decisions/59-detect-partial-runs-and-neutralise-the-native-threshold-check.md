---
type: Decision
status: draft
title: Detect Partial Runs and Neutralise the Native Threshold Check
description: Why a scoped Vitest run is detected via three independent signals and Vitest's native coverage-threshold check is neutralised and restored around it.
tags: [architecture, testing]
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: 2b47bd0a877d3fa708e8bbe61ef63a667fad9e698481153c7cbd5cbe03f94ec7
---

# Detect Partial Runs and Neutralise the Native Threshold Check

## Context

Vitest enforces `coverage.thresholds` against the whole-project denominator no matter how many test files ran: its coverage provider's `allTestsRun` flag gates only `autoUpdate`, and `checkThresholds` runs unconditionally from `reportCoverage`, which Vitest calls after every reporter's `onTestRunEnd`. A `vitest run foo.test.ts` therefore failed on coverage nothing in the run touched, and the plugin compounded it by detecting scoping only via a `projectFilter`, persisting every run with `test_runs.scoped = false`, and recomputing threshold violations on replay even when the report said `scoped`.

## Decision

A pure `isPartialRun({ filenamePattern, startedSpecCount, totalSpecCount, projectFilter })` (`packages/plugin/src/utils/is-partial-run.ts:41`) returns true on any of three signals: a non-empty Vitest `filenamePattern`, fewer specs started than `globTestSpecifications()` reports in total, or an explicit `projectFilter` (`packages/plugin/src/utils/is-partial-run.ts:43-45`). Three signals are needed because each filter surface is invisible to the others — a tags-only `run_tests` filter sets neither `filenamePattern` nor `projectFilter` and is caught only by the spec-count comparison. `projectFilter` is `AgentReporter`'s construction-time option (`packages/plugin/src/utils/is-partial-run.ts:14-23`), not the CLI `--project` flag: `packages/plugin/src/plugin.ts` never passes it when constructing the reporter, so in the production plugin path it is always `undefined` and a user's `--project` run is caught by the spec-count signal instead. Only a caller constructing `AgentReporter` directly (a test, a non-plugin embedding) ever sets it.

`AgentReporter` calls `isPartialRun` in `onTestRunEnd` (`packages/plugin/src/reporter.ts:1559`). A partial run routes coverage through `CoverageAnalyzer.processScoped` (`packages/plugin/src/reporter.ts:1892`), persists `scoped` honestly, emits no `ThresholdViolation`, skips the baseline/trend/threshold/target writes, and every renderer prints a "Coverage thresholds skipped: partial run" note through one shared formatter so a pass/fail verdict against a denominator that does not apply is never shown. Full-run output is byte-identical.

On a partial run the reporter deletes the metric keys (`lines`/`functions`/`branches`/`statements`) and every glob-pattern entry from `vitest.coverageProvider.options.thresholds` in place (`packages/plugin/src/reporter.ts:1592-1616`), keeping `perFile`, `autoUpdate`, and Vitest's internal `100` shorthand. `checkThresholds` reads that same object at report time, so with no metric keys left it has nothing to enforce. The deletion is not fire-and-forget: in `run` mode every `vitest.start` re-initialises the provider, but in watch mode the provider is created once and scoped reruns go through `rerunFiles` without re-initialising it, so a deleted key would stay gone for the rest of the session — one scoped rerun would silently disable thresholds for every subsequent full rerun. The reporter therefore snapshots every deleted key's original value into `neutralizedThresholdSnapshot` (`packages/plugin/src/reporter.ts:641`) as it deletes, and the next `onTestRunStart` unconditionally re-adds each snapshotted key onto the same provider-options object when it is still absent, then clears the map (`packages/plugin/src/reporter.ts:903-922`). A legitimately re-initialised provider (`run` mode) already carries its own fresh values and is left alone.

## Alternatives rejected

There is no supported API for suppressing threshold enforcement on a scoped run: `resolveOptions()` returns the options object but accepts no override, the reporter hook order is fixed, and `checkThresholds` has no per-run opt-out. Asking users to pass `--coverage.thresholds` overrides on every scoped invocation pushes the problem onto the caller; wrapping the provider couples to far more surface than the two fields actually at issue. Mutating the one object the plugin already has a handle on is the smallest intervention that fixes the observed failure. Making `checkThresholds` aware of partiality some other way would still require reaching into the same provider internals, since Vitest exposes no partial-run concept of its own.

## Consequences

This depends on two Vitest internals holding: the provider exposing `options.thresholds`, and `checkThresholds` reading it by reference at report time. A future Vitest refactor could silently make the neutralisation a no-op, in which case the symptom reverts to the original bug (a spurious threshold failure on a scoped run) rather than a crash — both the deletion and the restoration are guarded end to end (a missing provider, options, or thresholds shape is a no-op) and wrapped in try/catch. The snapshot-and-restore pair keeps the mutation bounded to a single run even when the provider outlives it (watch mode). The mutation is scoped to the in-process Vitest instance and never touches the user's config file.
