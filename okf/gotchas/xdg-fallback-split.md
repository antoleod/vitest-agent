---
type: Gotcha
title: XDG data-root fallback splits between the reporter and the hook routes
description: >-
  With XDG_DATA_HOME unset, the reporter/MCP route and the hook/sidecar route
  resolve two different fallback data roots, so they can silently open two
  different databases.
resource: ../../packages/engine/src/utils/resolve-data-path.ts
tags: [dx, architecture]
stale_after: "2027-03-12T00:00:00Z"
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: 227ce89e6cb20a8d17f45041890da47e72434d9bfdf510c8f2f14263b84815c2
sources:
  - id: resolve-data-path
    resource: ../../packages/engine/src/utils/resolve-data-path.ts
  - id: hook-paths
    resource: ../../packages/engine/src/programs/hook-paths.ts
  - id: path-resolution-live
    resource: ../../packages/engine/src/layers/PathResolutionLive.ts
---

# XDG data-root fallback splits between the reporter and the hook routes

A reader who traces `resolveDataPath` (used by the reporter and the MCP
server) and `resolveHookPaths` (used by the sidecar-family CLI subcommands
and the native sidecar binary) sees both build their XDG data root through
`@effected/xdg`'s `AppDirs`, and reasonably concludes the two front ends
share one `data.db` per project. **What is actually true:** the two call
sites configure `AppDirs` with different fallback directories when
`XDG_DATA_HOME` is unset, so they land in two different places.

`resolveDataPath` runs through `PathResolutionLive`, which builds
`AppDirs.layer({ namespace: "vitest-agent" })` with no `fallbackDir`
option[^resolve-data-path]. `AppDirs`'s own default fallback for that
shape is `~/.vitest-agent`. `resolveHookPaths`, by contrast, explicitly
passes `fallbackDir: ".local/share/vitest-agent"`[^hook-paths] — the XDG
spec's own default — specifically so the hook/sidecar route matches what
the sidecar family has "always used"[^hook-paths].

The consequence: with `XDG_DATA_HOME` set (the common case on Linux, and
anywhere a user or CI job has exported it), both routes resolve under
`$XDG_DATA_HOME/vitest-agent/<projectKey>/` and agree. Only when
`XDG_DATA_HOME` is unset do they diverge — the reporter and MCP server open
`~/.vitest-agent/<projectKey>/data.db` while every Claude Code plugin hook
and the `vitest-agent-sidecar` binary open
`~/.local/share/vitest-agent/<projectKey>/data.db`. A hook can write a
`tdd_artifacts` row that the MCP server's `DataReader` never sees, and vice
versa, with no error on either side — each route is internally consistent,
just consistent with a different file.

This is a known, deliberately preserved discrepancy rather than a bug
scheduled for a fix: aligning the two fallbacks would silently relocate
real installs' existing data. Anyone debugging "the hook wrote something
the MCP tool can't find" (or the reverse) on a machine without
`XDG_DATA_HOME` set should check both directories before assuming data
loss.

## Related

- [Decision 31 — Deterministic XDG Path Resolution](../decisions/31-deterministic-xdg-path-resolution.md)
- [Module: engine](../modules/engine.md)

[^resolve-data-path]: `../../packages/engine/src/utils/resolve-data-path.ts`
[^hook-paths]: `../../packages/engine/src/programs/hook-paths.ts`
