# @vercel/eval-framework

**Author:** judegao
**Date:** 2026-01-26

## Introduction

`@vercel/eval-framework` is an agent-agnostic evaluation framework for testing AI coding agents. It enables you to run the same test cases across different agent configurations (skills, CLAUDE.md files, models) and measure success rates.

The framework separates **what you're testing** (evals) from **how you're running tests** (configs), making it easy to compare different agent setups without duplicating test cases.

## Project Structure

```
my-evals/
├── agents/                    # Your agent definitions
│   └── claude.ts
├── configs/                   # Run configurations
│   └── skill-comparison.ts
├── evals/                     # Test cases
│   └── server-component/
│       ├── prompt.md          # Task description for the agent
│       ├── fixture/           # Starting project state
│       │   ├── app/page.tsx
│       │   └── package.json
│       └── page.test.ts       # Validation tests (optional)
└── results/                   # Output from eval runs
    └── skill-comparison/
        └── 2026-01-26T12:00:00/
            └── server-component/
                ├── baseline.json
                ├── withSkill.json
                └── comparison.json
```

## Defining Agents

Agents are functions that take a prompt and sandbox, then return a transcript.

The transcript is stored in the results folder for later investigation. When an eval fails, you can analyze the transcript to understand what the agent did, what tools it called, and where it went wrong. This is how you derive learnings and improve the agent.

```ts
// agents/claude.ts
import type { Agent, Transcript } from '@vercel/eval-framework'

type Agent = (prompt: string, sandbox: Sandbox) => Promise<Transcript>

export const claude: Agent = async (prompt, sandbox) => {
  await sandbox.exec('npm i -g @anthropic-ai/claude-code')
  await sandbox.exec(`claude --print --model opus -p "${prompt}"`)

  // Read transcript for storage in results
  const transcript = await sandbox.readFile('~/.claude/projects/.../transcript.jsonl')
  return JSON.parse(transcript)
}
```

For special setup (like installing a skill or copying a CLAUDE.md file), embed that setup directly in the agent:

```ts
// agents/claude-with-skill.ts
export const claudeWithSkill: Agent = async (prompt, sandbox) => {
  // Setup: install skill before running
  await sandbox.exec('npx @vercel/next-skill install')

  // Run agent
  await sandbox.exec('npm i -g @anthropic-ai/claude-code')
  await sandbox.exec(`claude --print --model opus -p "${prompt}"`)

  const transcript = await sandbox.readFile('~/.claude/projects/.../transcript.jsonl')
  return JSON.parse(transcript)
}
```

## Writing Configs

Configs determine which agent to run and how many times.

### Single Run

The simplest config runs each eval once:

```ts
// configs/default.ts
import { claude } from '../agents/claude'

export default {
  agent: claude,
}
```

### Best-of N Runs

Use `bestOf` to run until you find a pass (or hit the limit). This is useful when you want to know "can this agent ever pass this eval?"

```ts
// configs/best-of.ts
import { claude } from '../agents/claude'

export default {
  agent: claude,
  bestOf: 10,  // Stop early when a run passes
}
```

### Flakiness Testing

Use `runs` to run every time and report statistics. This is useful for measuring reliability.

```ts
// configs/flakiness.ts
import { claude } from '../agents/claude'

export default {
  agent: claude,
  runs: 10,  // Run all 10, report passRate, stddev, mean duration
}
```

### Comparing Variants

Use `variants` to compare different agent configurations:

```ts
// configs/skill-comparison.ts
import { claude } from '../agents/claude'
import { claudeWithSkill } from '../agents/claude-with-skill'

export default {
  variants: {
    baseline: claude,
    withSkill: claudeWithSkill,
  },
}
```

You can combine `variants` with `runs` or `bestOf`:

```ts
// configs/skill-flakiness.ts
export default {
  variants: {
    baseline: claude,
    withSkill: claudeWithSkill,
  },
  runs: 10,  // Run each variant 10 times
}
```

## Running Evals

```bash
# Run all evals with a config
npx eval configs/default.ts

# Run specific evals
npx eval configs/default.ts server-component

# Run evals matching a pattern
npx eval configs/flakiness.ts "server-*"
```

## Writing Tests

Test files validate that the agent made the correct changes. They're copied to the sandbox after the agent finishes, then executed inside the sandbox with `vitest run`.

Since tests run inside the sandbox (same directory as the fixture), file paths are relative to the project root:

```ts
// evals/server-component/page.test.ts
import { describe, it, expect } from 'vitest'
import { readFileSync } from 'fs'

describe('server component', () => {
  it('fetches data on the server', () => {
    // Runs inside sandbox, so paths are relative to project root
    const page = readFileSync('./app/page.tsx', 'utf-8')
    expect(page).toContain('async function')
    expect(page).toContain('fetch')
  })

  it('does not use client directive', () => {
    const page = readFileSync('./app/page.tsx', 'utf-8')
    expect(page).not.toContain("'use client'")
  })
})
```

## How Evals are Scored

An eval run is binary: **pass (1) or fail (0)**. No partial scores.

For an eval to pass, ALL of the following must succeed:

| Check | How it runs | Required |
|-------|-------------|----------|
| **Build** | `npm run build` | Yes - must be defined in package.json |
| **Lint** | `npm run lint` | Yes - must be defined in package.json |
| **Test** | `npm run test` (vitest) | Yes - if test files exist |

If `build` or `lint` scripts are not defined in package.json, the eval fails. This prevents false positives from missing validation.

The framework is **not tied to Next.js**—it works with any project that defines `build` and `lint` scripts in package.json.

## Results

Each eval run produces a result file with detailed information for investigation:

```json
// results/default/2026-01-26T12:00:00/server-component/run-1.json
{
  "eval": "server-component",
  "passed": false,
  "duration": 45200,
  "checks": {
    "build": { "passed": true, "duration": 12300, "output": "..." },
    "lint": { "passed": true, "duration": 2100, "output": "..." },
    "test": { "passed": false, "duration": 1800, "output": "...", "failures": ["renders product name"] }
  },
  "transcript": [
    { "type": "tool_use", "tool": "write_file", "path": "app/page.tsx", ... },
    { "type": "tool_use", "tool": "bash", "command": "npm install", ... },
    ...
  ],
  "metadata": {
    "agent": "claude",
    "model": "opus",
    "timestamp": "2026-01-26T12:00:00Z"
  }
}
```

The transcript is critical for improvement. When an eval fails, analyze the transcript to understand:
- What files did the agent create/modify?
- What commands did it run?
- Where did it go wrong?

This is how you derive learnings and improve agents or skills.

## CLI Output

### Single Run

When running a single eval, the CLI shows a simple pass/fail:

```
npx eval configs/default.ts server-component

server-component ✓ PASS (45.2s)
```

Or on failure:

```
server-component ✗ FAIL (38.1s)
  Build: ✓
  Lint:  ✓
  Test:  ✗ (1 failed)
    - renders product name

Results: results/default/2026-01-26T12:00:00/server-component/
```

### Best-of Report

```
npx eval configs/best-of.ts server-component

server-component (best of 10)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Result:    ✓ PASS
Attempts:  3 (stopped early)
Duration:  38.2s

Results: results/best-of/2026-01-26T12:00:00/server-component/
```

### Flakiness Report

```
npx eval configs/flakiness.ts server-component

server-component (10 runs)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pass rate:  7/10 (70%)
Mean:       45.2s
Stddev:     8.3s

Results: results/flakiness/2026-01-26T12:00:00/server-component/
```

### Variants Comparison

```
npx eval configs/skill-comparison.ts server-component

server-component
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            baseline    withSkill
Result      ✗ FAIL      ✓ PASS
Duration    38.2s       42.1s

Winner: withSkill

Results: results/skill-comparison/2026-01-26T12:00:00/server-component/
```

### Variants + Flakiness

```
npx eval configs/skill-flakiness.ts server-component

server-component (10 runs each)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            baseline    withSkill
Pass rate   4/10        9/10
Mean        48.2s       41.3s

Winner: withSkill (90%)

Results: results/skill-flakiness/2026-01-26T12:00:00/server-component/
```

## Sandbox

The sandbox provides an isolated environment for each eval run.

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
   - Copy test files to sandbox (if any)
   - Run `npm run build`
   - Run `npm run lint`
   - Run `npm run test` (if test files exist)
   - Record pass/fail (all must pass)
   - Save transcript and results to `results/`
3. Report summary

## Future

- `dimensions` for cartesian product comparisons (skill × model × sandbox)
