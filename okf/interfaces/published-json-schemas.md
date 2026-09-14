---
type: Interface
title: Published JSON Schema documents
description: The generated JSON Schema documents @vitest-agent/sdk ships and serves at vitest-agent.dev.
kind: config
resource: ../../packages/sdk/schemas
tags:
  - docs
  - release
  - compat
status: draft
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: 904e207e279597e785bca23afeb4caad0068734578d22d7e600acde94a7a6271
sources:
  - id: generate-schemas
    resource: ../../packages/sdk/scripts/generate-schemas.ts
  - id: run-report-file-schema-doc
    resource: ../../packages/sdk/schemas/run-report-file-1.0.0.json
  - id: sdk-package-json
    resource: ../../packages/sdk/package.json
---

# Interface: published JSON Schema documents

## What stays stable

`@vitest-agent/sdk` publishes generated, committed JSON Schema documents
under `packages/sdk/schemas/`. Today that set holds exactly one document:
`run-report-file-1.0.0.json`, the schema for
[Interface: report-files](report-files.md)'s `run.json`
envelope.[^run-report-file-schema-doc] `package.json`'s
`"./schemas/*.json"` export subpath lets a consumer resolve the document
offline without a network round trip.[^sdk-package-json]

## Where the document is served

Each document's `$id` is a `vitest-agent.dev` URL —
`RUN_REPORT_FILE_SCHEMA_URL` is
`https://vitest-agent.dev/schemas/run-report-file-1.0.0.json` for the
current document. The generator
(`packages/sdk/scripts/generate-schemas.ts`) builds one `SchemaTarget`
per output through `@effected/schemastore`'s `SchemaPipeline` and emits
the *same* document into two places from one source: the sdk package's
own `schemas/` directory (what ships to npm and what `./schemas/*.json`
resolves) and the website's public schemas directory (what the `$id` URL
resolves to once the docs site deploys).[^generate-schemas] These are two
generated targets, not a hand-copy — `packages/sdk/__test__/run-report-file-schema.test.ts`
imports both targets, so a drift between them, a stale document, or a
contract change without a version bump is a test failure rather than a
silent 404 of stale content.

## The release gate

The website copy must be live at its `$id` URL **before** `@vitest-agent/sdk`
publishes a version whose `RUN_REPORT_FILE_SCHEMA_URL` points at it —
otherwise a consumer's editor or validator resolves a schema `$id` that
404s. See [Runbook: release](../runbooks/release.md) for the ordering
this forces across the family's independent per-package releases.

## How a consumer references a document

- **Offline, from code.** `import schema from "@vitest-agent/sdk/schemas/run-report-file-1.0.0.json"` resolves the committed copy with no network access.
- **From a `run.json` file.** The file itself carries an optional
  `$schema` field pointing at the served URL, so an editor with JSON
  Schema support can validate it in place.
- **From `vitest-agent.config.toml`.** See
  [Interface: config-toml](config-toml.md) for the config file's own
  schema story — the TOML loader validates against
  `packages/sdk/src/schemas/Config.ts` directly rather than a published
  JSON Schema document, since no separate document is generated for it
  today.

A contract change to the underlying Effect Schema bumps the document's
version, `$id`, and filename together — `@effected/schemastore`'s
`block-versioned` policy fails the generation pipeline when it does not.
See [Decision 67](../decisions/67-report-files-are-a-versioned-public-contract.md)
for why the envelope carries its own version independent of any package
release.

[^generate-schemas]: `../../packages/sdk/scripts/generate-schemas.ts`
[^run-report-file-schema-doc]: `../../packages/sdk/schemas/run-report-file-1.0.0.json`
[^sdk-package-json]: `../../packages/sdk/package.json`
