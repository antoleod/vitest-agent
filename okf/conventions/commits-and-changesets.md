---
type: Convention
title: Commit and changeset discipline
description: "Every commit is a conventional commit with DCO signoff under silk's commitlint rules, and every branch that touches a package carries one changeset naming that package — the Claude Code plugin names @vitest-agent/claude-code-plugin, never @vitest-agent/plugin."
tags: [release, dx]
status: stable
stale_after: 2027-03-13T00:00:00Z
sources:
  - id: commitlint-config
    resource: ../../lib/configs/commitlint.config.ts
    title: Root commitlint config (CommitlintConfig.silk())
  - id: silk-commitlint-package
    resource: "npm:@savvy-web/silk/commitlint"
    title: "@savvy-web/silk's commitlint rule engine (type enum, TDD scope grammar, body constraints)"
  - id: changeset-config
    resource: ../../.changeset/config.json
  - id: husky-pre-commit
    resource: ../../.husky/pre-commit
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: 937f069afe816decd0c5f8f2067cb2053d3af019502974a1cbf4671449b1bcb8
---

# Commit and changeset discipline

## Every commit is a conventional commit with DCO signoff

`lib/configs/commitlint.config.ts` exports
`CommitlintConfig.silk()`[^commitlint-config] with no local overrides,
so every commit in this repository is checked against
`@savvy-web/silk`'s full commitlint rule set[^silk-commitlint-package]
via the `commit-msg` Husky hook. Two requirements apply to every
commit regardless of type:

- **Conventional commit format**: a `type(scope): subject` header using
  one of the enumerated types (`feat`, `fix`, `chore`, `ci`, `docs`,
  and so on).
- **DCO signoff**: a trailing `Signed-off-by: Name <email>` line.

Beyond the header shape, the silk rule set enforces body constraints
that are easy to violate without realizing it: a commit body line
capped at 300 characters, no inline code spans (backtick-fenced text)
in the body, and no double-underscore sequences in the body (a name
like `__test__` or a double-underscored MCP tool name trips the
markdown-in-body check and must be described in prose instead). Split
an overflowing bullet and drop backticks rather than fighting the
linter.

## The `tdd` commit type needs a scoped grammar

A commit of type `tdd` is rejected unless its scope matches
`tdd(<goalId>:<state>)` — a bare `tdd: ...` or a differently-shaped
scope fails the same silk rule set that enforces the type
enum[^silk-commitlint-package]. This scope grammar exists because the
TDD orchestration workstream commits once per red/green/refactor
transition and the scope is what makes that history greppable by goal
and phase.

## One changeset per package a branch touches

`.changeset/config.json` sets `updateInternalDependencies:
"patch"`[^changeset-config]: a changeset naming one package bumps only
that package, plus a patch ripple to its direct workspace dependents —
there is no lockstep "fixed" version group across the family. A branch
that touches `N` publishable packages needs `N` changesets (or one
changeset naming all `N`), never a single changeset that only names
the package the author happened to be thinking about.

`privatePackages: { tag: true, version: true }`[^changeset-config]
means every private workspace package — including
`@vitest-agent/claude-code-plugin` — still gets a version bump, a git
tag, and a GitHub Release from a changeset even though it is never
published to npm.

## Name `@vitest-agent/claude-code-plugin` for plugin-only changes

The Claude Code plugin at `plugins/claude-code/` versions through the
private `@vitest-agent/claude-code-plugin` tracking package. This is
not incidental bookkeeping: `.changeset/config.json`'s `changelog`
entry maps that exact package name's `versionFiles` to
`plugins/claude-code/.claude-plugin/plugin.json`'s `$.version`
field[^changeset-config] — a changeset naming a different package
never touches that manifest's version at all. Writing a changeset that
names `@vitest-agent/plugin` for a change that only touched
`plugins/claude-code/**` bumps the wrong package and forces a pointless
npm publish of the actual carrier package the family's consumers
install; the correct target for a plugin-only change is always
`@vitest-agent/claude-code-plugin`.

## Nothing in this pipeline bypasses lint-staged

The `pre-commit` hook[^husky-pre-commit] runs lint-staged over staged
files before `commit-msg` ever sees the message — Biome formats and
organizes imports, markdownlint-cli2 autofixes markdown, and
`tsgo --noEmit` blocks the commit on a real type error. None of these
autofixers touch the commit message itself; the commitlint checks
above are the only gate on message content.

See [Decision 36](../decisions/36-independent-per-package-release.md)
for why the family gave up a lockstep version group, and
[Decision 64](../decisions/64-claude-code-plugin-as-a-release-only-pnpm-workspace.md)
for why the plugin versions through a private tracking package rather
than being folded into `@vitest-agent/plugin`. See
[Runbook: Release](../runbooks/release.md) for the end-to-end release
procedure this discipline feeds.

[^commitlint-config]: ../../lib/configs/commitlint.config.ts
[^silk-commitlint-package]: npm:@savvy-web/silk/commitlint
[^changeset-config]: ../../.changeset/config.json
[^husky-pre-commit]: `.husky/pre-commit`
