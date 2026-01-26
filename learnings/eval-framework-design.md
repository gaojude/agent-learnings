# @vercel/eval-framework

**Author:** judegao
**Date:** 2026-01-26

## Introduction

`@vercel/eval-framework` is an agent-agnostic evaluation framework for testing AI coding agents. It enables you to run the same test cases across different agent configurations (skills, CLAUDE.md files, models) and measure success rates.

The framework separates **what you're testing** (evals) from **how you're running tests** (configs), making it easy to compare different agent setups without duplicating test cases.

## Getting Started

### Project Structure

```
my-evals/
├── agents/                    # Your agent definitions
│   └── claude.ts
├── configs/                   # Run configurations
│   └── skill-comparison.ts
└── evals/                     # Test cases
    └── server-component/
        ├── prompt.md          # Task description for the agent
        ├── fixture/           # Starting project state
        │   ├── app/page.tsx
        │   └── package.json
        └── page.test.ts       # Validation tests
```

### Running Evals

```bash
# Run all evals with a config
npx eval configs/skill-comparison.ts

# Run specific evals
npx eval configs/skill-comparison.ts server-component

# Run evals matching a pattern
npx eval configs/flakiness.ts "server-*"
```

## Defining Agents

Agents are functions that take a prompt and sandbox, then return a transcript. The framework doesn't care how the agent works internally—it just needs access to the sandbox to make changes.

```ts
// agents/claude.ts
import type { Agent } from '@vercel/eval-framework'

type Agent = (prompt: string, sandbox: Sandbox) => Promise<Transcript>

export const createClaude = (model = 'opus'): Agent =>
  async (prompt, sandbox) => {
    await sandbox.exec('npm i -g @anthropic-ai/claude-code')
    await sandbox.exec(`claude --print --model ${model} -p "${prompt}"`)

    // Return transcript from ~/.claude
    const transcript = await sandbox.readFile('~/.claude/transcript.jsonl')
    return JSON.parse(transcript)
  }

export const claude = createClaude('opus')
export const sonnet = createClaude('sonnet')
```

### Composing Agents

You can wrap agents to add behavior like installing skills or copying CLAUDE.md files:

```ts
// agents/with-skill.ts
import type { Agent } from '@vercel/eval-framework'
import { claude } from './claude'

const withPreHook = (cmd: string) => (agent: Agent): Agent =>
  async (prompt, sandbox) => {
    await sandbox.exec(cmd)
    return agent(prompt, sandbox)
  }

// Create variants
export const claudeWithSkill = withPreHook('npx @vercel/next-skill install')(claude)
export const claudeWithClaudeMd = withPreHook('cp ./templates/nextjs.md CLAUDE.md')(claude)
```

## Writing Configs

Configs determine which agents to run and how many times. They're decoupled from test cases, so you can run any config against any set of evals.

### Basic Config

```ts
// configs/default.ts
import { claude } from '../agents/claude'

export default {
  agent: claude,
  timeout: 300_000,
}
```

### Comparing Variants

Use `variants` to run the same evals with different agent configurations:

```ts
// configs/skill-comparison.ts
import { claude, claudeWithSkill, claudeWithClaudeMd } from '../agents'

export default {
  variants: {
    baseline: claude,
    withSkill: claudeWithSkill,
    withClaudeMd: claudeWithClaudeMd,
  },
}
```

### Measuring Flakiness

Use `runs` to run each eval multiple times and report statistics:

```ts
// configs/flakiness.ts
import { claude } from '../agents'

export default {
  agent: claude,
  runs: 10,  // Run all 10, report passRate, stddev, mean duration
}
```

### Finding a Passing Run

Use `bestOf` to run until you find a pass (or hit the limit):

```ts
// configs/best-of.ts
import { claudeWithSkill } from '../agents'

export default {
  agent: claudeWithSkill,
  bestOf: 10,  // Stop early when a run passes
}
```

### Combining Variants with Runs

```ts
// configs/skill-flakiness.ts
import { claude, claudeWithSkill } from '../agents'

export default {
  variants: {
    baseline: claude,
    withSkill: claudeWithSkill,
  },
  runs: 10,  // Run each variant 10 times, report flakiness per variant
}
```

```ts
// configs/skill-best-of.ts
import { claude, claudeWithSkill } from '../agents'

export default {
  variants: {
    baseline: claude,
    withSkill: claudeWithSkill,
  },
  bestOf: 10,  // Find best run per variant
}
```

## Writing Tests

Test files are copied to the sandbox after the agent finishes, then executed with `vitest run`. Tests validate that the agent made the correct changes.

```ts
// evals/server-component/page.test.ts
import { describe, it, expect } from 'vitest'
import { readFileSync } from 'fs'

describe('server component', () => {
  it('fetches data on the server', () => {
    const page = readFileSync('app/page.tsx', 'utf-8')
    expect(page).toContain('async function')
    expect(page).toContain('fetch')
  })

  it('renders product name', () => {
    const page = readFileSync('app/page.tsx', 'utf-8')
    expect(page).toContain('<h1')
  })

  it('does not use client directive', () => {
    const page = readFileSync('app/page.tsx', 'utf-8')
    expect(page).not.toContain("'use client'")
  })
})
```

The framework also runs `next build` and `next lint` automatically. An eval passes only if build, lint, and tests all succeed.

## Sandbox

The sandbox provides an isolated environment for each eval run. The framework handles creating and destroying sandboxes.

```ts
interface Sandbox {
  exec(cmd: string): Promise<{ stdout: string; stderr: string; exitCode: number }>
  readFile(path: string): Promise<string>
  writeFile(path: string, content: string): Promise<void>
}
```

Providers: local, Docker, Vercel Sandbox.

## Execution Flow

1. Framework reads config and discovers evals
2. For each eval × variant × run:
   - Create sandbox
   - Copy `fixture/` to sandbox
   - Run `agent(prompt, sandbox)`
   - Copy test files to sandbox
   - Run `next build`, `next lint`, `vitest run`
   - Record pass/fail
3. Report results

## Reports

### Single Run

```
server-component
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Status:   ✓ PASS
Build:    ✓ (12.3s)
Lint:     ✓ (2.1s)
Tests:    ✓ (1.8s)
Duration: 45.2s
```

### Variants Comparison

```
npx eval configs/skill-comparison.ts server-component

server-component
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
              baseline    withSkill    withClaudeMd
Status        ✗ FAIL      ✓ PASS       ✓ PASS
Duration      38.2s       42.1s        40.5s

Winner: withSkill
```

### Flakiness Report (`runs: 10`)

```
npx eval configs/flakiness.ts server-component

server-component (10 runs)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pass rate:    7/10 (70%)
Mean:         45.2s
Stddev:       8.3s
Fastest:      32.1s
Slowest:      58.4s
```

### Variants + Flakiness (`variants` + `runs: 10`)

```
npx eval configs/skill-flakiness.ts server-component

server-component (10 runs each)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
              baseline    withSkill    withClaudeMd
Pass rate     4/10        9/10         7/10
Mean          48.2s       41.3s        44.8s
Stddev        12.1s       5.2s         8.9s

Winner: withSkill (90% pass rate)
```

### Best-of Report (`bestOf: 10`)

```
npx eval configs/skill-best-of.ts server-component

server-component (best of 10)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
              baseline    withSkill    withClaudeMd
Best run      ✗ FAIL      ✓ PASS       ✓ PASS
Attempts      10          3 ←early     6
Duration      --          38.2s        42.1s

Winner: withSkill (passed on attempt 3)
```

## Future

- `dimensions` for cartesian product comparisons (skill × model × sandbox)
