---
type: Limitation
title: Only one project's reporter instance processes the shared coverage map
description: "Vitest fires one onCoverage per process against a single global CoverageMap, but a multi-project config constructs one AgentReporter per project; only the reporter for the project with the most collected test modules runs CoverageAnalyzer against it, so coverage never attributes to a specific project."
bounds: ../modules/plugin.md
tags: [architecture, observability]
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: 1e8820865667688fe2a86ed259334de3ed323cbc315e8d1dddc00886d01bbc69
sources:
  - id: oncoverage
    resource: ../../packages/plugin/src/reporter.ts
  - id: dedup
    resource: ../../packages/plugin/src/reporter.ts
  - id: baselines
    resource: ../../packages/plugin/src/reporter.ts
---

# Only one project's reporter instance processes the shared coverage map

Vitest's coverage provider calls `onCoverage` once per test run with a
single istanbul `CoverageMap` that spans every project in the run — it is
not scoped per project. `AgentReporter.onCoverage` stashes that value as
instance state for use in the later `onTestRunEnd` pass.[^oncoverage] A
multi-project Vitest config constructs one `AgentReporter` instance per
project (each filtered to its own modules via `projectFilter`), so
without a dedup rule every one of those instances would process — and
persist — the same whole-workspace `CoverageMap`.

**Condition.** A Vitest config with more than one project, coverage
enabled.

**Symptom.** `AgentReporter.onTestRunEnd` counts collected test modules
per project name, picks the project with the most modules as
`primaryProject`, and only that reporter instance's `isFirstProject`
check passes and calls `CoverageAnalyzer.process` /
`processScoped`.[^dedup] Every other project's reporter instance
persists no `coverage_reports` row of its own for that run. A consumer
querying per-project coverage sees it attached to whichever project
happened to have the most test modules that run, not to the project the
coverage actually measures — coverage in this family is a
whole-workspace concept, never a per-project one, regardless of which
project's name ends up owning the persisted row.

**Why this is acceptable.** Istanbul's `CoverageMap` genuinely has no
project boundary — files are covered by whichever test exercised them,
irrespective of which Vitest project ran that test — so there is no
correct per-project split to compute in the first place. Picking one
consistent owner avoids duplicate `coverage_reports` rows and duplicate
threshold-violation events for the same underlying data.

**What a fix would take.** A `coverage_reports` schema keyed on the run
rather than a project name (or a `NULL`/`__global__` project sentinel
enforced at the schema level, matching the `__global__` key already used
for baselines[^baselines]) would let every caller stop reading
project-attributed coverage as though it meant something project-scoped.
That is a migration, not a reporter-side fix.

[^oncoverage]: `../../packages/plugin/src/reporter.ts:1421`
[^dedup]: `../../packages/plugin/src/reporter.ts:1870-1894`
[^baselines]: `../../packages/plugin/src/reporter.ts:1864-1865`
