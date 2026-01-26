# @vercel/eval-framework

**Author:** judegao
**Date:** 2026-01-26

## Overview

Agent-agnostic eval framework. Tests skills across different harnesses (Next.js, Nuxt, etc.) with composable agent definitions.

Key ideas:
- Agents are functions, composed via FP modifiers
- Tests run outside sandbox, interrogate via API
- Configs select agent + eval strategy, decoupled from test cases

## Structure

```
agents/                         # Agent definitions (FP style)
  claude.ts
  modifiers.ts
configs/                        # How to run (which agent, how many runs)
  skill-comparison.ts
  flakiness.ts
evals/                          # What to test (pure test cases)
  server-component/
    prompt.md                   # Task for agent
    fixture/                    # Starting state → copied to sandbox
    page.test.ts                # Assertions (runs on host)
```

## Agent

Agents are functions: `(prompt, sandbox) → result`

```ts
type Agent = (prompt: string, sandbox: Sandbox) => Promise<RunResult>

// agents/claude.ts
export const createClaude = (model = 'opus'): Agent =>
  async (prompt, sandbox) => {
    await sandbox.exec('npm i -g @anthropic-ai/claude-code')
    return sandbox.exec(`claude --print --model ${model} -p "${prompt}"`)
  }

export const claude = createClaude('opus')
export const sonnet = createClaude('sonnet')
```

## Modifiers

Wrap agents to add behavior (preHooks, skills, etc.):

```ts
// agents/modifiers.ts
export const withPreHook = (cmd: string) => (agent: Agent): Agent =>
  async (prompt, sandbox) => {
    await sandbox.exec(cmd)
    return agent(prompt, sandbox)
  }

// Convenience
export const withSkill = (pkg: string) => withPreHook(`npx ${pkg} install`)
export const withClaudeMd = (path: string) => withPreHook(`cp ${path} CLAUDE.md`)
```

## Composition

Compose agents for different variants:

```ts
// agents/index.ts
import { claude } from './claude'
import { withSkill, withClaudeMd, pipe } from './modifiers'

export const claudeWithSkill = withSkill('@vercel/next-skill')(claude)

export const claudeWithBoth = pipe(
  claude,
  withSkill('@vercel/next-skill'),
  withClaudeMd('./templates/nextjs.md'),
)
```

## Config

Configs pick agents and run strategy. Decoupled from test cases.

```ts
// configs/skill-comparison.ts
import { claude, claudeWithSkill } from '../agents'

export default {
  variants: {
    baseline: { agent: claude },
    withSkill: { agent: claudeWithSkill },
  },
  bestOf: 10,
}
```

### `runs` vs `bestOf`

| Key | Behavior |
|-----|----------|
| `runs: N` | Run all N, report flakiness (passRate, stddev) |
| `bestOf: N` | Run up to N, early exit on pass, report best |

## Test

Tests run on host. Sandbox stays pure (exactly what agent produced).

```ts
// evals/server-component/page.test.ts
import { test, expect } from '@vercel/eval-framework/test'

test('renders button', async ({ sandbox }) => {
  const page = await sandbox.readFile('app/page.tsx')
  expect(page).toContain('<button')
})

test('builds successfully', async ({ sandbox }) => {
  const { exitCode } = await sandbox.exec('npm run build')
  expect(exitCode).toBe(0)
})

test('agent was efficient', async ({ trace }) => {
  expect(trace.commands.filter(c => c.includes('build'))).toHaveLength(1)
})
```

## Sandbox

Interface for test assertions. Providers: local, Docker, Vercel Sandbox.

```ts
interface Sandbox {
  readFile(path: string): Promise<string>
  writeFile(path: string, content: string): Promise<void>
  exists(path: string): Promise<boolean>
  glob(pattern: string): Promise<string[]>
  exec(cmd: string): Promise<{ stdout: string; stderr: string; exitCode: number }>
  fetch(route: string): Promise<string>
}
```

## CLI

Config is primary arg. Filters are positional (default: all evals).

```bash
npx eval configs/default.ts                     # all evals
npx eval configs/skill-comparison.ts "server-*" # glob filter
npx eval configs/flakiness.ts 001 002 003       # specific evals
```

## Flow

```
1. Read config (agent, variants, runs/bestOf)
2. For each eval:
   a. fixture/ → sandbox
   b. agent(prompt, sandbox)
   c. tests interrogate sandbox from host
   d. record pass/fail
3. Report results
```

## Future

- `dimensions` for cartesian product (skill × model × sandbox)
