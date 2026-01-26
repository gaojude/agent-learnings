# @vercel/eval-framework

**Author:** judegao
**Date:** 2026-01-26

## Model

```
config(agent, variants, runs) + evals(prompt, fixture, test) → results
```

## Structure

```
agents/
  claude.ts
  modifiers.ts
configs/
  skill-comparison.ts
evals/
  server-component/
    prompt.md
    fixture/
    page.test.ts
```

## Agent

```ts
type Agent = (prompt: string, sandbox: Sandbox) => Promise<RunResult>

// agents/claude.ts
export const createClaude = (model = 'opus'): Agent =>
  async (prompt, sandbox) => {
    await sandbox.exec('npm i -g @anthropic-ai/claude-code')
    return sandbox.exec(`claude --print --model ${model} -p "${prompt}"`)
  }

export const claude = createClaude('opus')
```

## Modifiers

```ts
// agents/modifiers.ts
export const withPreHook = (cmd: string) => (agent: Agent): Agent =>
  async (prompt, sandbox) => {
    await sandbox.exec(cmd)
    return agent(prompt, sandbox)
  }

export const withSkill = (pkg: string) => withPreHook(`npx ${pkg} install`)
export const withClaudeMd = (path: string) => withPreHook(`cp ${path} CLAUDE.md`)
```

## Composition

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

## `runs` vs `bestOf`

| Key | Behavior |
|-----|----------|
| `runs: N` | Run all N, report flakiness stats |
| `bestOf: N` | Run up to N, early exit on pass |

## Test

Tests run on host, interrogate sandbox via API:

```ts
// evals/server-component/page.test.ts
import { test, expect } from '@vercel/eval-framework/test'

test('renders', async ({ sandbox }) => {
  const page = await sandbox.readFile('app/page.tsx')
  expect(page).toContain('<button')
})

test('builds', async ({ sandbox }) => {
  const { exitCode } = await sandbox.exec('npm run build')
  expect(exitCode).toBe(0)
})
```

## Sandbox

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

Providers: local, Docker, Vercel Sandbox.

## CLI

```bash
npx eval configs/default.ts                    # all evals
npx eval configs/skill-comparison.ts "server-*" # filtered
```

## Flow

```
1. fixture/ → sandbox
2. agent(prompt, sandbox)
3. tests interrogate sandbox
4. report
```

## Future

- `dimensions` for cartesian product (skill × model × sandbox)
