---
type: Decision
title: Report Files Are a Versioned Public Contract
description: The run.json envelope Vitest 5's createReport writes carries its own schemaVersion independent of any package version, published as a JSON Schema document, because it is read by code with no dependency on @vitest-agent/sdk.
status: draft
tags:
  - architecture
  - dx
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: c44f2d064eccd67dc62cb3f6b40e9bb41ac1e8c5d195cf753445802f653a0541
sources:
  - id: run-report-file-schema
    resource: ../../packages/sdk/src/schemas/RunReportFile.ts
  - id: generate-schemas-script
    resource: ../../packages/sdk/scripts/generate-schemas.ts
  - id: published-schema-copy
    resource: ../../website/docs/public/schemas/run-report-file-1.0.0.json
  - id: report-writer
    resource: ../../packages/plugin/src/utils/report-writer.ts
  - id: plugin-report-option
    resource: ../../packages/plugin/src/plugin.ts
  - id: rendered-output-type
    resource: ../../packages/sdk/src/formatters/types.ts
---

# Report Files Are a Versioned Public Contract

## Context

Vitest 5 added `vitest.createReport(scope)`, a supported way for a
reporter to write files into `.vitest/<scope>/`. vitest-agent uses it for
two files: `run.json` for machine readers — a Claude Code hook, a CI step,
an agent that never saw the terminal — and a human-facing summary. A
machine reader of `run.json` may have no dependency on
`@vitest-agent/sdk` at all, so the file's shape cannot be "whatever
`AgentReport` happens to encode to this month" and drift silently with a
package version bump.

## Decision

`run.json` is `{ $schema, schemaVersion: 1, generatedAt, reports:
AgentReport[] }` — an envelope carrying its own contract version
independent of any npm package version.[^run-report-file-schema] It is
published as a JSON Schema document at
`https://vitest-agent.dev/schemas/run-report-file-1.0.0.json`, generated
from the Effect Schema by
`packages/sdk/scripts/generate-schemas.ts`[^generate-schemas-script] into
two committed targets: the sdk's own copy that ships to npm, and the docs
site's `public/schemas/` copy that the `$id` URL
resolves to.[^published-schema-copy] `RenderedOutput` (the type every
formatter returns) is a discriminated union whose `report` member carries
a flat `filename` alongside `content` and `contentType`, distinct from the
`stdout` / `file` / `github-summary` members that share one
shape.[^rendered-output-type] The plugin owns writing report files:
`utils/report-writer.ts` creates the `createReport` handle lazily on the
first report-targeted output — `createReport` mkdirs eagerly and
synchronously, so a run that emits no report output leaves no directory
behind — never calls `clean()` (it would wipe a prior shard's output and
is a no-op under `--merge-reports` anyway), rejects `/`, `\`, `.`, and
`..` in both the filename and the scope, and writes failures to stderr
rather than failing the run.[^report-writer] `AgentPlugin({ report })`
defaults report files on for the `agent` and `ci` executors and off for a
human at a terminal; `report: false` disables them entirely, and `report:
{ scope }` renames the directory, with `"vitest-agent"` as the default
scope.[^plugin-report-option]

## Alternatives rejected

- **Reuse the existing `file` target on `RenderedOutput`** for report
  output. Rejected because `file` has always been the reserved no-op with
  no path convention — giving report output its own discriminated-union
  member keeps the reporter out of path resolution entirely and lets
  Vitest's own `createReport` own cleanup, sharding, and merge behavior.
- **Version `run.json`'s shape implicitly through `@vitest-agent/sdk`'s
  own package version.** Rejected because a reader with no dependency on
  the sdk package — a shell hook, a CI step written in another language —
  has no way to observe a package version at all; an explicit
  `schemaVersion` field on the envelope is the only contract such a reader
  can check.
- **Ship the JSON Schema from only one location** (the sdk package or the
  docs site, not both). Rejected because the `$id` URL must resolve to a
  live document for schema tooling to fetch it, while the sdk package
  needs its own copy to ship with npm installs that never touch the
  website — committing both and generating them from the same Effect
  Schema keeps them from drifting relative to each other.

## Consequences

A contract change to `run.json` must bump the schema version, the `$id`
URL, and the filename together — the docs site's schema pipeline treats a
mismatch there as an error under its block-versioned policy, so a
one-sided bump fails before it reaches a consumer. Report files are
machine-facing and independent of console mode: an `agent`-mode run that
prints nothing to stdout still writes them, and the `.vitest/` directory
belongs in a consumer's `.gitignore` rather than being treated as ordinary
build output.

## Related

- [Interface: report-files](../interfaces/report-files.md)
- [Module: plugin](../modules/plugin.md)

[^run-report-file-schema]: `../../packages/sdk/src/schemas/RunReportFile.ts:21,55-57`
[^generate-schemas-script]: `../../packages/sdk/scripts/generate-schemas.ts`
[^published-schema-copy]: `../../website/docs/public/schemas/run-report-file-1.0.0.json`
[^rendered-output-type]: `../../packages/sdk/src/formatters/types.ts:17-24`
[^report-writer]: `../../packages/plugin/src/utils/report-writer.ts:43-46,54-60,74-80,83-112`
[^plugin-report-option]: `../../packages/plugin/src/plugin.ts:400-410`
