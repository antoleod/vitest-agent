---
type: Decision
title: Cap Inline Attachment Bodies on Stored Bytes, Not the Reported Size
description: DataStoreLive persists an attachment body inline only when both the caller-reported byteSize and the actual stored-string length clear a 64 KiB cap, so a caller cannot smuggle an oversized row past a self-reported number.
status: draft
tags:
  - architecture
  - performance
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: dc42d3e85356058453f133a79375329f629b24b1521ed7587e91937a7f77c6c2
sources:
  - id: data-store-service
    resource: ../../packages/engine/src/services/DataStore.ts
  - id: data-store-live
    resource: ../../packages/engine/src/layers/DataStoreLive.ts
  - id: migration-0002
    resource: ../../packages/engine/src/migrations/0002_test_artifacts.ts
---

# Cap Inline Attachment Bodies on Stored Bytes, Not the Reported Size

## Context

A test attachment can be a multi-megabyte trace. Vitest has already
copied a file-based attachment into `.vitest/attachments/` and rewritten
the descriptor's `path` before it reaches this system, so `data.db` never
needs those bytes duplicated into a row. A small inline body — a diff, a
log excerpt, an HTML snippet — is worth persisting directly, though,
because the `.vitest/attachments/` path may be cleaned away later and an
agent reading over MCP has no filesystem access to the runner that
produced it anyway.

## Decision

`INLINE_ATTACHMENT_BODY_CAP_BYTES` is `64 * 1024` (64 KiB).[^data-store-service]
`DataStoreLive.writeAttachments` stores a body inline only when
`Math.max(storedBytes, att.byteSize) <= INLINE_ATTACHMENT_BODY_CAP_BYTES`,
where `storedBytes` is `Buffer.byteLength(att.body, "utf8")` — the actual
size of the string that would land in the row, not the caller's reported
`byteSize`.[^data-store-live] `byte_size` is recorded verbatim on every
attachment row regardless of whether the body clears the cap, so a
dangling `.vitest/attachments/` path stays describable even after the
directory is cleaned; `path` is recorded exactly as Vitest resolved it and
no file is ever copied into the database.[^data-store-live] The
`byte_size` and `body_encoding` columns this decision relies on were added
to the `attachments` table by migration
`0002_test_artifacts.ts`.[^migration-0002]

## Alternatives rejected

- **Gate inline storage on the caller's reported `byteSize` alone.**
  Rejected because that number is caller-supplied and unverified — a
  producer that under-reports or zero-reports its own attachment size
  could smuggle an arbitrarily large string into the row while still
  passing the cap check.
- **Gate on the stored string's length alone, ignoring the reported
  size.** Rejected because a base64-encoded body's stored string is 4/3
  the size of the payload it represents, so gating only on the stored
  string under-charges exactly the encoding this system is told to expect
  by default.
- **Cap the body length and silently truncate rather than dropping it
  entirely.** Rejected — a truncated body reads as complete to anything
  that decodes it, silently corrupting whatever format the attachment
  carries (a truncated base64 string does not decode to a truncated
  image; it fails to decode at all). Recording only the descriptor when
  the cap is exceeded is honest about what was and was not kept.

## Consequences

An attachment over the cap persists as a bare descriptor —
`content_type`, `path`, `byte_size`, and no `body` — which is exactly what
a reader needs to go fetch the bytes some other way. `body_encoding` is
stored alongside any body that does get kept, so a reader always knows
whether the string is base64 or UTF-8 without guessing from content.
Requiring both bars to clear before storing inline makes the row's real
on-disk cost, not a caller's self-report, the thing this cap actually
bounds.

## Related

- [Decision 69 — MCP Attachment Bodies Are Opt-In and Budgeted](./69-mcp-attachment-bodies-are-opt-in-and-budgeted.md)
- [Decision 66 — Migration 0002 Drops the Dead Table and ALTERs the Live Ones](./66-migration-0002-drops-the-dead-table-and-alters-the-live-ones.md)
- [DataModel: sqlite-schema](../models/sqlite-schema.md)

[^data-store-service]: `../../packages/engine/src/services/DataStore.ts:201,208,212`
[^data-store-live]: `../../packages/engine/src/layers/DataStoreLive.ts:284-299`
[^migration-0002]: `../../packages/engine/src/migrations/0002_test_artifacts.ts:51-57`
