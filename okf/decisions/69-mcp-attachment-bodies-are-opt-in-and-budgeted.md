---
type: Decision
title: MCP Attachment Bodies Are Opt-In and Budgeted
description: The test tool's annotations and artifacts actions return attachment descriptors by default and only include inline bodies when the caller passes a cumulative maxBytes budget, because a per-attachment cap says nothing about the total size of one tool response.
status: draft
tags:
  - mcp
  - performance
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: 2143d43a9f0527a4f3a4d236d33351039ae29dff1fc4e9340f9b28fa7f1c9ebc
sources:
  - id: mcp-test-tool
    resource: ../../packages/mcp/src/tools/test.ts
  - id: data-store-cap
    resource: ../../packages/engine/src/services/DataStore.ts
---

# MCP Attachment Bodies Are Opt-In and Budgeted

## Context

The `test` tool's `annotations` and `artifacts` actions return every
annotation or artifact recorded for one test in the latest run, each
carrying its own attachments. The 64 KiB persistence cap on an inline
attachment body is per attachment, so a test with many inline attachments
could still flood an agent's context window on a single tool call even
though no individual attachment exceeds the persistence cap.

## Decision

Both actions return attachment descriptors by default — `contentType`,
`path`, `byteSize` — with no `body` field.[^mcp-test-tool] An inline
`body` comes back only when the caller passes `maxBytes`, a non-negative
integer total byte budget for every body in the response, defaulting to
`0`.[^mcp-test-tool] `applyBodyBudget` walks the attachments in order and
charges each body its recorded `byteSize` (falling back to the stored
string's own length when the row predates migration 0002 and carries no
`byteSize`); a body that would push the running total past the budget is
dropped along with its `bodyEncoding`, while the descriptor half —
`contentType`, `path`, `byteSize` — always survives regardless of
budget.[^mcp-test-tool]

## Alternatives rejected

- **A per-body cap on the response, mirroring the persistence-layer
  cap.** Rejected because a per-body cap bounds one attachment and says
  nothing about the response as a whole; the real constraint an agent
  cares about is the total size of the tool result it has to read, which
  only a cumulative budget expresses.
- **Return bodies by default up to some fixed response-wide ceiling with
  no caller control.** Rejected because a fixed default ceiling still
  charges every caller for bytes most calls never need — the count and
  descriptor list already tell an agent everything required to decide
  whether fetching bodies is worth the tokens, so paying for bytes should
  be a deliberate second step rather than baked into the default call.
- **Charge each body its stored string length instead of its recorded
  `byteSize`.** Rejected for the same reason the persistence-layer cap
  gates on both bars (Decision 68): a base64 body's stored length is 4/3
  its real payload size, so charging the stored length rather than the
  recorded `byteSize` would under-count the true cost against the
  caller's budget.

## Consequences

The count and the full descriptor list are identical no matter what
`maxBytes` is set to, so an agent can always see what exists before
deciding whether to pay for the bytes. Defaulting `maxBytes` to `0` means
the cheap call — descriptors only — is the default call, and every caller
that wants bodies has to say so explicitly with an explicit budget.

## Related

- [Decision 68 — Cap Inline Attachment Bodies on Stored Bytes](./68-cap-inline-attachment-bodies-on-stored-bytes.md)
- [Interface: mcp-tools](../interfaces/mcp-tools.md)

[^mcp-test-tool]: `../../packages/mcp/src/tools/test.ts:92-119,400-428`
