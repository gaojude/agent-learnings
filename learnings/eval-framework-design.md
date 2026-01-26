# Eval Framework Design

**Date:** 2026-01-26
**Author:** judegao
**Project:** @vercel/eval-framework

## Overview

A zero-config, agent-agnostic framework for evaluating AI coding agents across any frontend framework. Inspired by Next.js DX principles.

## Core Model

```
config + test cases + sandbox + agent → transformed state → validation
```

- **Agents act on environments via tools** (file writes, commands), not just stdout
- **Tests run outside the sandbox**, interrogating it via API
- **Sandbox stays pure** - exactly what the agent produced
- **Configs are decoupled from test cases** - same config applies to many tests

## File Structure

```
configs/                        # How to run evals
  default.ts
  flakiness.ts
  skill-comparison.ts
  agent-comparison.ts

evals/                          # What to test (pure test cases)
  add-button/
    prompt.md                   # Agent instructions
    fixture/                    # Starting state → copied to sandbox
      app/page.tsx
      package.json
    page.test.ts                # Assertions (runs on host, not in sandbox)
  server-component/
    prompt.md
    fixture/
    page.test.ts
```

| Path | Purpose | Copied to sandbox? |
|------|---------|-------------------|
| `configs/*.ts` | Run configuration | No |
| `evals/*/fixture/` | Input state for agent | Yes (before agent) |
| `evals/*/prompt.md` | Agent instructions | No |
| `evals/*/*.test.ts` | Eval assertions | No (runs on host) |

## CLI

Config file is the primary argument. Filters are positional args after config (default: all).

```bash
npx eval configs/default.ts                          # All evals
npx eval configs/default.ts server-component         # One eval
npx eval configs/default.ts server-component client-component  # Multiple
npx eval configs/default.ts "server-*"               # Glob pattern
npx eval configs/flakiness.ts "server-*" "client-*"  # Multiple globs
```

## Config Design

Configs define "how" to run evals. Test cases define "what" to test.

### Simple: Single Run

```ts
// configs/default.ts
export default {
  agent: 'claude',
  sandbox: 'vercel',
  timeout: 300_000,
}
```

### Flakiness: Run N Times

```ts
// configs/flakiness.ts
// Run 10 times, report pass rate + variance
export default {
  runs: 10,
}
```

### Comparison: Variants

```ts
// configs/skill-vs-baseline.ts
export default {
  variants: {
    baseline: {},
    withSkill: { preHook: 'npx @vercel/next-skill install' },
  },
}
```

### Comparison with Multiple Runs

```ts
// configs/skill-vs-claudemd-vs-baseline.ts
// Smart defaults: with variants, runs implies best-of behavior
// (parallel execution, early exit on pass, report best)
export default {
  variants: {
    baseline: {},
    withClaudeMd: { preHook: 'cp ./templates/nextjs.md CLAUDE.md' },
    withSkill: { preHook: 'npx @vercel/next-skill install' },
  },
  runs: 10,
}
```

### Agent Comparison

```ts
// configs/agent-comparison.ts
export default {
  variants: {
    claude: { agent: 'claude' },
    codex: { agent: 'codex' },
    cursor: { agent: 'cursor' },
  },
  runs: 5,
}
```

### Model Comparison

```ts
// configs/model-comparison.ts
export default {
  variants: {
    opus: { agent: 'claude', model: 'opus' },
    sonnet: { agent: 'claude', model: 'sonnet' },
  },
}
```

## Smart Defaults

Behavior is inferred from config shape:

| Config | Behavior |
|--------|----------|
| `{ runs: 10 }` | Flakiness mode: run all 10, report stats |
| `{ variants: {...} }` | Comparison mode: run each variant once |
| `{ variants: {...}, runs: 10 }` | Best-of mode: parallel, early exit on pass, report best per variant |

## Execution Flow

```
1. Setup:     fixture/ ──────────────────▶ sandbox
2. PreHook:   config.preHook runs (if defined)
3. Agent:     agent.run(prompt, sandbox)  ▶ sandbox (modified)
4. PostHook:  config.postHook runs (if defined)
5. Validate:  tests interrogate sandbox from outside
6. Report:    pass/fail + traces
```

## Test Design

Tests run on the host, access sandbox via API:

```ts
import { test, expect } from '@vercel/eval-framework/test'

// File assertions
test('button exists', async ({ sandbox }) => {
  const page = await sandbox.readFile('app/page.tsx')
  expect(page).toContain('<button')
})

// Command assertions
test('build passes', async ({ sandbox }) => {
  const result = await sandbox.exec('npm run build')
  expect(result.exitCode).toBe(0)
})

// HTTP assertions
test('page renders', async ({ sandbox }) => {
  const html = await sandbox.fetch('/')
  expect(html).toContain('<button')
})

// Trace assertions (process goals)
test('agent was efficient', async ({ trace }) => {
  expect(trace.commands.filter(c => c.includes('build'))).toHaveLength(1)
})

// Model-assisted grading
test('follows best practices', async ({ sandbox }) => {
  const code = await sandbox.readFile('app/page.tsx')
  const score = await grade(code, { rubric: '...', passingScore: 6 })
  expect(score.total).toBeGreaterThanOrEqual(6)
})
```

## Agent Interface

```ts
interface Agent {
  run(prompt: string, sandbox: Sandbox): Promise<Trace>
}
```

Agents are pluggable: Claude Code, Codex, Cursor, custom. Framework doesn't care how code is generated.

## Sandbox Interface

```ts
interface Sandbox {
  // File operations
  readFile(path: string): Promise<string>
  writeFile(path: string, content: string): Promise<void>
  exists(path: string): Promise<boolean>
  glob(pattern: string): Promise<string[]>

  // Command execution
  exec(cmd: string): Promise<{ stdout: string; stderr: string; exitCode: number }>

  // HTTP (sandbox handles server internally)
  fetch(route: string): Promise<string>
}
```

Sandbox providers: local, Docker, Vercel Sandbox.

## Output Structure

Results mirror the config + eval structure:

```
results/
  skill-vs-baseline/
    2026-01-26T12:00:00/
      server-component/
        baseline.json
        withSkill.json
        comparison.json
      client-component/
        ...
  flakiness/
    2026-01-26T13:00:00/
      server-component/
        run-01.json
        run-02.json
        ...
        summary.json   # { passRate: 0.8, mean: 45s, stddev: 5s }
```

## Example Output

```
skill-vs-claudemd-vs-baseline @ server-component
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
                  baseline    withClaudeMd    withSkill
Best run          ✗ FAIL      ✓ PASS          ✓ PASS
Pass rate         2/10        7/10            10/10 ←early exit
Mean duration     45s         52s             38s

Winner: withSkill (100% pass rate, fastest)
```

## Key Decisions

1. **Tests outside sandbox** - Sandbox stays pure, tests interrogate from host
2. **Configs decoupled from test cases** - One config applies to many tests
3. **Agent-agnostic** - Any agent that can manipulate a sandbox
4. **Framework-agnostic** - Works with Next.js, Nuxt, Remix, etc.
5. **Smart defaults** - Behavior inferred from config shape (flakiness vs comparison vs best-of)
6. **Config is primary CLI arg** - `npx eval <config> [filters...]`

## Architecture

```
@vercel/eval-framework/
  core/
    runner.ts           # Orchestration
    config.ts           # Config parsing + smart defaults
    sandbox/
      local.ts
      docker.ts
      vercel.ts
    agents/
      claude.ts
      codex.ts
      custom.ts
  test/
    index.ts            # Test utilities + context
    grade.ts            # Model-assisted grading
  cli.ts
```

## Validation Layers

1. **Deterministic** - File exists, contains string, command exits 0
2. **Behavioral** - HTTP response contains X, dev server starts
3. **Process** - Agent didn't loop, used correct tools, was efficient
4. **Qualitative** - Model-assisted grading against rubric
