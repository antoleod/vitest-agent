---
type: Decision
title: Deterministic XDG Path Resolution
description: The data path is a deterministic function of workspace identity under XDG's data directory, resolved through one precedence chain for the reporter/MCP route and a second, divergent one for the hook/sidecar route.
status: draft
tags:
  - architecture
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: 1a91d8035f8d17731f560558829917ccd7b9f82b00ed4311f59aba06a687bc4e
sources:
  - id: engine-resolve-data-path
    resource: ../../packages/engine/src/utils/resolve-data-path.ts
  - id: engine-resolve-project-key-from-cwd
    resource: ../../packages/engine/src/utils/resolve-project-key-from-cwd.ts
  - id: engine-path-resolution-live
    resource: ../../packages/engine/src/layers/PathResolutionLive.ts
  - id: engine-hook-paths
    resource: ../../packages/engine/src/programs/hook-paths.ts
  - id: engine-project-identity-live
    resource: ../../packages/engine/src/layers/ProjectIdentityLive.ts
  - id: engine-project-identity
    resource: ../../packages/engine/src/services/ProjectIdentity.ts
  - id: engine-resolve-workspace-key
    resource: ../../packages/engine/src/utils/resolve-workspace-key.ts
---

# Deterministic XDG Path Resolution

## Context

An earlier resolver walked the filesystem looking for an existing build
artifact (a `data.db` nested under `node_modules/.vite/vitest/<hash>/...`)
and fell back to a literal path when no artifact existed yet. That made the
data path depend on filesystem state — "does this artifact exist right
now?" — rather than on the identity of the workspace, so the reporter and
the MCP server could independently compute different paths for the same
project; that form is retired.

## Decision

The data path is a deterministic function of the workspace's identity:
`$XDG_DATA_HOME/vitest-agent/<workspaceKey>/data.db`.
`resolveDataPath` (`packages/engine/src/utils/resolve-data-path.ts:59-87`) —
used by the reporter and the MCP server — resolves `<workspaceKey>` through
a precedence chain: (1) a caller-supplied `options.cacheDir` (the plugin's
programmatic `reporter.cacheDir`), highest precedence
(`packages/engine/src/utils/resolve-data-path.ts:64-67`); (2) `cacheDir`
from `vitest-agent.config.toml`
(`packages/engine/src/utils/resolve-data-path.ts:73-76`); (3) `projectKey`
from that same TOML, normalized via `normalizeWorkspaceKey`
(`packages/engine/src/utils/resolve-data-path.ts:82`); (4) failing those,
`resolveProjectKeyFromCwd` — a `package.json` read anchored at the caller's
directory that prefers a canonicalized `repository.url`
(`host__path`) and falls back to the normalized `name`
(`packages/engine/src/utils/resolve-project-key-from-cwd.ts:33-59`). The
XDG data root itself comes from `@effected/xdg`'s `AppDirs.layer({
namespace: "vitest-agent" }).pipe(Layer.provide(Xdg.layer))`
(`packages/engine/src/layers/PathResolutionLive.ts:19`), which defaults to
`~/.vitest-agent` when `XDG_DATA_HOME` is unset — `AppDirs`' own fallback,
not the `~/.local/share/vitest-agent` path documented as the spec default.

**A separate route exists for the sidecar-family hooks.**
`resolveHookPaths` (`packages/engine/src/programs/hook-paths.ts:164-185`)
builds its own `AppDirs` layer with an explicit
`fallbackDir: ".local/share/vitest-agent"`
(`packages/engine/src/programs/hook-paths.ts:58,111-116`) so that, with
`XDG_DATA_HOME` unset, the hook/sidecar route lands at
`~/.local/share/vitest-agent/<projectKey>/` — the XDG spec default the
sidecar has always used — while the reporter/MCP route above lands at
`~/.vitest-agent/<projectKey>/`. This is a live, acknowledged split between
the two routes, not a bug scheduled for a specific fix; see
[xdg-fallback-split](../gotchas/xdg-fallback-split.md).

**Identity resolution also diverges by route, and only one route fails
loud.** The hook/sidecar route's `ProjectIdentity` service candidates
(explicit → TOML → git remote → `package.json` repository → `package.json`
name) fail with `ProjectIdentityNotResolvableError` when every candidate is
empty (`packages/engine/src/layers/ProjectIdentityLive.ts:148`,
`packages/engine/src/services/ProjectIdentity.ts:124`) — this is the
fail-loud contract this decision was written to guarantee. The
reporter/MCP route's `resolveProjectKeyFromCwd`, however, never fails: with
no reachable `package.json` it falls back to the cwd's basename, and with
no non-empty basename segment at all, to the literal string
`"anonymous-project"` (`packages/engine/src/utils/resolve-project-key-from-cwd.ts:52-59`).
The reporter/MCP route therefore does not raise
`WorkspaceRootNotFoundError` or any other typed error on missing identity
today — that error type exists on `resolveWorkspaceKey`
(`packages/engine/src/utils/resolve-workspace-key.ts:31-47`), which is
defined but has no caller in the current tree. The delta from this
decision's original fail-loud intent for the reporter/MCP route is a real
drift, reported below.

**Why XDG:** the database is workspace-scoped user state, not
project-build-output — it does not belong under `node_modules` (wiped by
`rm -rf node_modules`) or inside the project tree (clutters `git status`).
XDG's "user data" category is the correct semantic match, and
`@effected/xdg` honors `XDG_DATA_HOME` cross-platform with a defined
fallback.

**Why workspace-name keying instead of a path hash:** two checkouts
(worktrees) of the same repository share history under a name-keyed path,
where a path hash would diverge them; the database follows project identity
across a disk move rather than filesystem coordinates; `ls
~/.local/share/vitest-agent/` (or `~/.vitest-agent/`) shows readable package
names rather than opaque hashes, useful for manual inspection and
debugging; and a fork that renames its `package.json` `name` gets its own
database while a fork that keeps the same name shares one — overridable
either way via `projectKey`.

**Why TOML for the override file:** TOML's distinction between strings and
bare identifiers reads more naturally for path-like configuration than
JSON's everything-is-a-string, and `@effected/config-file`'s `TomlCodec`
integrates directly with Effect Schema decoding.

## Alternatives rejected

- **Filesystem-artifact probing** (the retired predecessor): rejected
  because the path it produced depended on whether a build artifact
  happened to already exist, letting two processes (reporter, MCP server)
  disagree about where the database lived.
- **Path-hash keying** (hash of the absolute project path): rejected in
  favor of workspace-name keying — see the worktree-consistency and
  disk-move-resilience rationale above; a path hash would defeat both.
- **JSON for the override config file**: rejected in favor of TOML for
  readability of path-like values and direct `@effected/config-file`
  integration.

## Consequences

- A workspace-name collision — two unrelated projects sharing the same root
  `package.json` `name` with no `projectKey` override — resolves both to
  the same `<workspaceKey>` and they share a database. The `projectKey`
  config override and the human-readable XDG layout are the only
  mitigations; nothing detects the collision automatically.
- The reporter/MCP route's silent `"anonymous-project"` fallback means a
  project with no discoverable `package.json` name and no `.git` remote
  gets a database at a fixed, shared path rather than a load or startup
  error — a behavior at odds with this decision's original "fail loud on
  missing identity" intent, which only the hook/sidecar route actually
  implements today.
- Any new hook or reporter code path that needs the data directory must
  choose explicitly between the reporter/MCP `AppDirs` default
  (`~/.vitest-agent`) and the hook/sidecar `fallbackDir` override
  (`~/.local/share/vitest-agent`) — reusing the wrong one silently produces
  a database at the wrong path for that caller.

## Drift reported

The old decision's "Why fail-loud on missing workspace identity" section
described `WorkspaceRootNotFoundError` as the error the reporter/MCP route
raises on missing identity. In the current tree, `resolveWorkspaceKey` (the
function that raises `WorkspaceRootNotFoundError`) has no caller, and the
route actually used by `resolveDataPath` —
`resolveProjectKeyFromCwd` — never fails, falling back silently to a cwd
basename or `"anonymous-project"`. The fail-loud behavior described survives
only on the separate hook/sidecar route, via
`ProjectIdentityNotResolvableError`. This concept documents both routes as
they exist today rather than the single unified route the old decision
assumed.

## Related

- [xdg-fallback-split](../gotchas/xdg-fallback-split.md)
