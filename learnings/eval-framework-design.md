# @vercel/eval-framework

**Author:** judegao
**Date:** 2026-01-26

## Introduction

`@vercel/eval-framework` is an opinionated evaluation framework for testing AI coding agents on Node.js projects.

**Assumptions:** npm, vitest, claude-code CLI

## Project Structure

```
my-evals/
├── evals/
│   └── add-button/           # This folder IS the fixture
│       ├── src/
│       │   └── App.tsx
│       ├── package.json
│       ├── PROMPT.md         # What to ask the agent
│       └── EVAL.ts           # How to validate (hidden from agent)
├── configs/                  # Optional run configurations
│   └── with-skill.ts
└── results/
```

The eval folder **is** the fixture—a normal Node.js project. Two special files:
- `PROMPT.md` — the prompt sent to the agent
- `EVAL.ts` — validation tests (agent never sees this)

## Writing Evals

### PROMPT.md

The prompt for the agent. Just markdown:

```md
Add a logout button to the header.

Requirements:
- Button should be in the top right
- Clicking it should clear the session and redirect to /login
```

### EVAL.ts

Validation tests. Can import from the fixture since it runs after the agent:

```ts
// evals/add-button/EVAL.ts
export const scripts = ["build", "lint"]

import { files } from '@vercel/eval-framework'
import { describe, it, expect } from 'vitest'

describe('logout button', () => {
  it('exists somewhere in codebase', () => {
    const hasLogout = Object.values(files).some(c => /logout/i.test(c))
    expect(hasLogout).toBe(true)
  })

  it('has click handler', () => {
    const hasHandler = Object.values(files).some(c => /onClick.*logout/i.test(c))
    expect(hasHandler).toBe(true)
  })
})
```

### Exports

| Export | Type | Description |
|--------|------|-------------|
| `scripts` | `string[]` | npm scripts that must exit 0 (optional) |

### The `files` Object

The framework provides `files`—a `Record<string, string>` mapping file paths to contents. Populated with the agent's output (excluding `node_modules`, `.git`, lockfiles, etc.) before tests run.

```ts
import { files } from '@vercel/eval-framework'

files['src/App.tsx']              // get specific file
Object.keys(files)                // list all paths
Object.values(files)              // all contents
```

Use standard vitest assertions. The framework doesn't abstract vitest—you get full access to all its features.

## Execution Flow

1. Copy eval folder to sandbox (excluding `PROMPT.md`, `EVAL.ts`)
2. Run `npm install`
3. Run agent with contents of `PROMPT.md`
4. Populate `files` with output
5. Inject `EVAL.ts`
6. Run vitest
7. Run `scripts` (if defined)
8. Result: **0 or 1**

## Scoring

An eval is binary: **pass (1) or fail (0)**. No partial scores.

For an eval to pass:
- All vitest tests must pass
- All `scripts` must exit 0

## CLI

The framework ships with a built-in claude-code agent:

```bash
# Single run (uses claude-code by default)
npx eval

# Run specific eval
npx eval add-button

# Best of N (stop on first pass)
npx eval --best-of 5

# Flakiness testing (run all, report stats)
npx eval --runs 10

# With options
npx eval --model opus --runs 10
```

## Configs

For reusable setups, create a config file:

```ts
// configs/with-skill.ts
export default {
  model: 'opus',
  skills: ['@vercel/next-skill'],
  runs: 10,
}
```

```bash
npx eval --config configs/with-skill.ts
```

### Custom Agent

For non-claude-code agents, define a custom agent in the config:

```ts
// configs/custom.ts
export default {
  agent: async (prompt, sandbox) => {
    // your agent logic
    await sandbox.exec(`my-agent "${prompt}"`)
    return { transcript: [...] }
  },
  runs: 10,
}
```

## CLI Output

### Single Run

```
npx eval add-button

add-button ✓ PASS (45.2s)
```

### Flakiness Report

```
npx eval --runs 10

add-button (10 runs)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pass rate:  7/10 (70%)
Mean:       45.2s
Stddev:     8.3s
```

### Config Comparison

```
npx eval --config a.ts --config b.ts --runs 10

add-button (10 runs each)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            config-a    config-b
Pass rate   4/10        9/10
Mean        48.2s       41.3s

Winner: config-b (90%)
```

## Results

Each run produces a result file:

```json
{
  "eval": "add-button",
  "passed": true,
  "duration": 45200,
  "scripts": {
    "build": { "passed": true, "duration": 12300 },
    "lint": { "passed": true, "duration": 2100 }
  },
  "tests": {
    "passed": true,
    "failures": []
  },
  "transcript": [...],
  "metadata": {
    "config": "with-skill.ts",
    "model": "opus",
    "timestamp": "2026-01-26T12:00:00Z"
  }
}
```

The transcript is critical for improvement. When an eval fails, analyze it to derive learnings and improve the agent.

## Sandbox

```ts
interface Sandbox {
  exec(cmd: string): Promise<{ stdout: string; stderr: string; exitCode: number }>
  readFile(path: string): Promise<string>
  writeFile(path: string, content: string): Promise<void>
}
```

Providers: local, Docker, Vercel Sandbox.
