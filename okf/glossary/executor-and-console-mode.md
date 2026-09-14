---
type: Glossary
title: Executor vs. console mode
description: >-
  "Executor" is a detected fact about who is running the tests
  (human/agent/ci); "console mode" is a resolved output-behavior value
  looked up per executor. The two use the literal string "agent" for
  unrelated things.
tags: [dx, architecture]
generated:
  by: okfit/claude-code
  at: 2026-09-14T02:24:39Z
  body_sha256: 9b82a5198852ba8a4a063dbbde43d175d0991af0155265cdf49b450108b65738
sources:
  - id: executor-schema
    resource: ../../packages/sdk/src/schemas/Common.ts
  - id: console-mode-schemas
    resource: ../../packages/sdk/src/schemas/Common.ts
  - id: options-console-field
    resource: ../../packages/sdk/src/schemas/Options.ts
  - id: resolve-console-mode
    resource: ../../packages/plugin/src/plugin.ts
---

# Executor vs. console mode

## Executor: detected

`Executor` is a closed three-value schema — `"human" | "agent" | "ci"`
(`packages/sdk/src/schemas/Common.ts:126`). `AgentPlugin.configureVitest`
detects the environment (`EnvironmentDetector`) and maps it to one of these
three values through `envToExecutor(env)` (`packages/plugin/src/plugin.ts:254`,
called at `packages/plugin/src/plugin.ts:402`). Nothing in `AgentPluginOptions`
lets a user set the executor directly — it is always inferred from the
process environment, never configured.

## Console mode: resolved

`ConsoleMode` is the union of three *different*, executor-scoped literal
sets: `HumanConsoleMode` (`"passthrough" | "silent" | "stream" | "agent"`),
`AgentConsoleMode` (`"passthrough" | "silent" | "agent"`), and
`CiConsoleMode` (`"passthrough" | "silent" | "ci-annotations"`)
(`packages/sdk/src/schemas/Common.ts:71-100`). A user configures per-executor
preferences via `AgentPluginOptions.console` — an object with optional
`human` / `agent` / `ci` keys (`packages/sdk/src/schemas/Options.ts:22-24`,
`:72`) — and `resolveConsoleMode(options, executor, env)`
(`packages/plugin/src/plugin.ts:121`) looks up `console.<executor>`,
validates a `VITEST_AGENT_CONSOLE` env override against the executor's own
literal set, and falls back to a per-slot default (`human` →
`"passthrough"`, `agent` → `"agent"`, `ci` → `"passthrough"`;
`packages/plugin/src/plugin.ts:146-152`).

## The trap

The literal string `"agent"` means two unrelated things depending on which
axis it appears on:

- As an **executor**, `"agent"` means "an LLM agent is driving this process"
  — detected, not chosen.
- As a **console mode** (valid only for the `human` and `agent` executor
  slots), `"agent"` means "render the markdown-flavored final-frame string
  instead of passing through to Vitest's own reporters" — a resolved output
  behavior, and it is one of several modes any executor's slot can carry
  (a human executor can also be given `console.human = "agent"`).

Reading `console: { agent: "agent" }` as redundant, or assuming the
`console mode` value always matches the detected `executor`, is the
recurring misreading: a human executor with `console.human = "agent"` still
gets agent-flavored rendering, and an agent executor is free to set
`console.agent = "passthrough"` and get none.

Two consequences follow purely from the resolved console mode value, not
the executor: any mode other than `"passthrough"` strips Vitest's own
console reporters and suppresses its native coverage table
(`ownsStdout`, `packages/plugin/src/plugin.ts:163`), and when no `reporter`
option is supplied the plugin injects `DefaultVitestAgentReporter`, which
branches internally on the resolved mode.

See [Decision 37](../decisions/37-per-executor-console-matrix-streaming-reporter-tap.md)
and [Interface agent-plugin-options](../interfaces/agent-plugin-options.md).
