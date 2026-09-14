---
type: Invariant
title: Ranked layering — every workspace edge points strictly downward
description: Every workspace package has a declared rank, every dependency edge points to a strictly lower rank, cli and mcp never depend on each other, and the carrier is the only rank-5 package — enforced structurally, not by convention.
tags: [architecture]
resource: ../../packages/plugin/__test__/workspace-layering.test.ts
sources:
  - id: layering-test
    resource: ../../packages/plugin/__test__/workspace-layering.test.ts
  - id: workspace-graph
    resource: ../../packages/plugin/__test__/utils/workspace-graph.ts
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: 26be5049d2902d367c6d8e78b25a5f628580c8d6824e7ca50d8d59e24cc7e9b4
---

# Ranked layering — every workspace edge points strictly downward

## Property

Every workspace package (each `packages/*`, `plugins/*`, `website`,
`playground`, and the root `vitest-agent` package) is assigned exactly one
integer rank, and every `dependencies` / `devDependencies` /
`peerDependencies` / `optionalDependencies` edge from one workspace package
to another must point to a strictly lower rank than the one it originates
from[^workspace-graph]. Two ranks additionally carry named constraints: the
two rank-4 packages, `@vitest-agent/cli` and `@vitest-agent/mcp`, never
depend on each other, and `@vitest-agent/plugin` is the sole rank-5
package — the only package anything else in the family may not depend
on[^layering-test].

```text
1  @vitest-agent/sdk
2  @vitest-agent/ui, @vitest-agent/sidecar-{darwin-arm64,linux-arm64,linux-x64,win32-x64}
3  @vitest-agent/engine, @vitest-agent/reporter, @vitest-agent/sidecar
4  @vitest-agent/cli, @vitest-agent/mcp
5  @vitest-agent/plugin (the carrier)
—  root vitest-agent (dev-only; depends on plugin alone)
```

The declared table also carries `@vitest-agent/claude-code-plugin`,
`docs`, and `playground` at rank 6, and the root `vitest-agent` package at
rank 7, so every workspace member has a home even where it sits outside
the publishable family[^workspace-graph].

## Mechanism

`packages/plugin/__test__/utils/workspace-graph.ts` reads every
workspace's `package.json` under `packages/`, `plugins/`, `website`, and
`playground`, plus the root manifest, and builds a directed graph whose
edges are every `dependencies` / `devDependencies` / `peerDependencies` /
`optionalDependencies` entry that names another workspace package. It
exports `LAYER_RANKS`, the hand-maintained `Record<string, number>` that
is the single source of truth for what rank each package occupies[^workspace-graph].

`packages/plugin/__test__/workspace-layering.test.ts` runs four
assertions over that graph[^layering-test]:

1. Every discovered package name is a key in `LAYER_RANKS` — a package
   with no declared rank fails immediately rather than silently passing
   the edge check with an `undefined` rank.
2. Every edge's target rank is strictly less than its source rank
   (`LAYER_RANKS[e.to] >= LAYER_RANKS[n.name]` is a violation).
3. `@vitest-agent/cli`'s edge list never contains `@vitest-agent/mcp`, and
   `@vitest-agent/mcp`'s edge list never contains `@vitest-agent/cli` —
   checked by name, not by rank, since both sit at rank 4 and the
   strictly-lower-rank check alone would not catch a same-rank edge
   between the two front ends.
4. The graph is acyclic: a Kahn's-algorithm topological sort starting
   from every zero-in-degree node must consume every node in the graph.
   A cycle anywhere — even one the rank check missed because of a
   `LAYER_RANKS` typo — leaves nodes unconsumed and fails the count
   comparison.

The test lives in the carrier's test tree specifically because the
carrier already depends on every other family package, so it is the one
location a single test file can see the whole graph, and because
root-level test files are not discovered by `classifyTestPath` — see
[Invariant: Test-path classification](./test-path-classification.md).

It lives alongside, but is distinct from, the four `boundaries.test.ts`
files: this test checks the shape of the dependency graph between
packages, while boundaries checks what each package's own source is
permitted to import or read — see
[Invariant: Package boundaries](./package-boundaries.md).

## What a refactor would have to break

Adding a new workspace package without an entry in `LAYER_RANKS` fails
assertion 1 immediately — there is no default rank a forgotten package
falls back to. Introducing a dependency edge in the wrong direction (for
example, `@vitest-agent/sdk` importing something from `@vitest-agent/ui`)
fails assertion 2 the moment that edge is added to a `package.json`,
independent of whether the corresponding TypeScript import actually
exists — the test reads manifests, not source. Adding a workspace
dependency from `@vitest-agent/cli` on `@vitest-agent/mcp` (or the
reverse) fails assertion 3 even though both are rank 4 and neither
technically violates the strictly-lower-rank rule. And restructuring the
graph so that two packages mutually depend on each other — even
indirectly, through a longer cycle — fails assertion 4's topological sort,
regardless of what any individual rank number says.

Because the check reads `package.json` manifests rather than TypeScript
imports, it cannot by itself catch a source file that imports another
package without a corresponding manifest dependency; it only proves that
every *declared* dependency edge respects the rank order. It is one half
of the carrier's structural guarantee — see
[Decision 70: Carrier Pattern and Ranked Layering](../decisions/70-carrier-pattern-and-ranked-layering.md)
for why the graph is shaped this way, [Module: workspace](../modules/workspace.md)
for the repository-wide layout the ranks apply to, and
[Glossary: Carrier](../glossary/carrier.md) for what makes
`@vitest-agent/plugin`'s rank-5 position load-bearing rather than
incidental.

[^layering-test]: ../../packages/plugin/__test__/workspace-layering.test.ts
[^workspace-graph]: ../../packages/plugin/__test__/utils/workspace-graph.ts
