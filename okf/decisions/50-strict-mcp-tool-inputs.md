---
type: Decision
status: draft
title: Strict MCP Tool Inputs
description: Every served MCP tool input rejects unknown keys at every object level instead of silently widening the query.
tags: [architecture, mcp]
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: ee5b2927891573e07e3ef3f441a783d9f7547e102f3c866bdf84b8bd9beeb65e
---

# Strict MCP Tool Inputs

## Context

Effect's `McpServer.toolkit` decodes tool arguments with the library
default `onExcessProperty: "ignore"`, so an unknown key is silently
stripped before the handler ever sees the payload
(`packages/mcp/src/register-toolkit.ts:1-6`). Two failure modes share
that root cause. A caller that misspells a filter parameter gets a
successful result computed from a wider filter set than it asked for —
`run_tests` would run the entire workspace and report green while the
agent believed it had scoped the run. And a served schema that simply
forgot to declare a parameter the handler accepts is indistinguishable,
from the wire, from one that ignored it on purpose. An agent cannot
detect either case from the response alone.

## Decision

Every served tool input is strict, and the rule applies per object
*level*, not per tool: an unknown key is rejected at the top level,
inside nested objects, inside array elements, and inside the union
branch a discriminant selects, with an error naming both the offending
key(s) — path-qualified — and the accepted params at that level.

The mechanism lives entirely in `registerStrictToolkit`
(`packages/mcp/src/register-toolkit.ts:405-411`), which every tool
registers through instead of Effect's own `McpServer.toolkit`.
`strictifyJsonSchema` (`register-toolkit.ts:120-166`) deep-copies each
tool's generated JSON Schema and sets `additionalProperties: false` on
every object node that declares `properties` or is a bare object with
no combinator, and rewrites a top-level union of discriminated object
shapes (an `action` / `kind` field shared by every member, detected by
`findDiscriminant`, `register-toolkit.ts:59-69`) into
`{ type: "object", oneOf, "x-discriminator" }` so the served schema
still satisfies MCP's object-root requirement. `inlineRootRefs`
(`register-toolkit.ts:101-109`) inlines a `$ref` root — what Effect
emits for any schema carrying an `identifier` annotation — before the
object checks run, so identified schemas register and list correctly.

Before decoding, `collectUnknownKeys` (`register-toolkit.ts:197-266`)
walks the raw payload against that served (strict) schema, recursing
through `properties`, `oneOf` / `anyOf` (via `selectMember`,
`register-toolkit.ts:177-189`), `allOf` (merged), `items`, and
`prefixItems`, and collects every level that carries a key the schema
does not declare. The `handle` callback registered on
`registry.addTool` (`register-toolkit.ts:349-393`) runs this walk first
and fails with `McpSchema.InvalidParams` (`formatUnknownKeys`,
`register-toolkit.ts:272-279`) before the built toolkit's own handler is
ever invoked — a rejected call never reaches a `DataReader` /
`DataStore` call.

Its retired zod-mechanics predecessor — a hand-synced `z.strictObject`
registration per tool — is superseded by this schema-walk approach;
see [Invariant: Strict Tool Inputs](../invariants/strict-tool-inputs.md)
for the enforcement contract this decision produces.

## Alternatives rejected

- **Partial adoption (strict on some tools, default-lenient on others):**
  rejected because a surface where unknown-key rejection is per-tool luck
  teaches an agent nothing it can rely on — the whole point is that a
  rejection is a promise, not a maybe.
- **Relying on Effect's own decode to reject excess properties:** not
  available — `McpServer.toolkit`'s decode step uses
  `onExcessProperty: "ignore"` unconditionally, so this had to be
  implemented as a pre-decode payload walk against the served schema
  rather than a decode-option flip.

## Consequences

- A misspelled or forgotten parameter now fails loudly with the
  offending key and the accepted set named, instead of silently running
  a wider query and reporting success.
- Every new tool must register through `registerStrictToolkit`
  (`packages/mcp/src/toolkit.ts`), never `McpServer.toolkit` directly,
  or it loses strict-input enforcement silently.
- `RunTestsOk` echoes the resolved filter set on a required
  `scope: { project, files, tags }` field as the success-path
  counterpart to rejection: one field distinguishes "ran exactly what I
  asked" from "ran everything," with no inference from summary counts
  needed for the one tool (`run_tests`) whose scope narrowing cannot be
  fully validated by a schema walk alone.

## Related

- [Invariant: Strict Tool Inputs](../invariants/strict-tool-inputs.md)
- [Decision 71 — Effect-Native MCP Server](./71-effect-native-mcp-server.md)
