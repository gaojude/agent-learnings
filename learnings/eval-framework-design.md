# Eval Framework Design

**Date:** 2026-01-26
**Author:** judegao
**Project:** @vercel/eval-framework

## Overview

A zero-config, agent-agnostic framework for evaluating AI coding agents across any frontend framework. Inspired by Next.js DX principles.

## Core Model

```
prompt + fixture + sandbox + agent → transformed state → validation
```

- **Agents act on environments via tools** (file writes, commands), not just stdout
- **Tests run outside the sandbox**, interrogating it via API
- **Sandbox stays pure** - exactly what the agent produced

## File Structure

```
evals/
  add-button/
    prompt.md           # Agent instructions
    fixture/            # Starting state → copied to sandbox
      app/page.tsx
      package.json
    page.test.ts        # Assertions (never enters sandbox)
    eval.config.ts      # Optional overrides
```

| Path | Purpose | Copied to sandbox? |
|------|---------|-------------------|
| `fixture/` | Input state for agent | Yes (before agent) |
| `prompt.md` | Agent instructions | No |
| `*.test.ts` | Eval assertions | No (runs on host) |
| `eval.config.ts` | Settings | No |

## Execution Flow

```
1. Setup:     fixture/ ──────────────────▶ sandbox
2. Agent:     agent.run(prompt, sandbox)  ▶ sandbox (modified)
3. Validate:  tests interrogate sandbox from outside
4. Report:    pass/fail + traces
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

  // HTTP (starts dev server internally)
  fetch(route: string): Promise<string>
  serve(cmd: string, opts: { port: number }): Promise<string> // returns URL
}
```

Sandbox providers: local, Docker, Vercel Sandbox.

## CLI

```bash
npx @vercel/eval                    # Run all evals
npx @vercel/eval add-button         # Run specific eval
npx @vercel/eval --agent codex      # Swap agent
npx @vercel/eval --sandbox docker   # Use Docker isolation
npx @vercel/eval --json             # Machine-readable output
```

## Progressive Disclosure

**Level 1 - Zero config:** Just `fixture/`, `prompt.md`, `*.test.ts`

**Level 2 - Config:**
```ts
// eval.config.ts
export default {
  agent: 'codex',
  sandbox: 'docker',
  timeout: 300_000,
}
```

**Level 3 - Programmatic:**
```ts
import { createEval } from '@vercel/eval-framework'

const result = await createEval({
  prompt: 'Add a submit button',
  fixture: './my-app',
  agent: myCustomAgent,
  sandbox: createDockerSandbox(),
})
```

## Key Decisions

1. **Tests outside sandbox** - Sandbox stays pure, tests interrogate from host
2. **Agent-agnostic** - Any agent that can manipulate a sandbox
3. **Framework-agnostic** - Works with Next.js, Nuxt, Remix, etc.
4. **Convention over config** - File structure is the API
5. **E2E is optional** - Most validation is file/command assertions; full Playwright is progressive enhancement

## Architecture

```
@vercel/eval-framework/
  core/
    runner.ts           # Orchestration
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
