# @vercel/eval-framework

**Author:** judegao
**Date:** 2026-01-26

## Introduction

`@vercel/eval-framework` is an opinionated evaluation framework for testing AI coding agents on Node.js projects.

**Assumptions:** npm, vitest

## Project Structure

```
my-evals/
├── evals/
│   └── add-button/           # This folder IS the fixture
│       ├── src/
│       │   └── App.tsx
│       ├── package.json
│       └── __eval__.ts       # Hidden from agent, injected after
└── results/
```

The eval folder **is** the fixture—a normal Node.js project. The only special file is `__eval__.ts`, which contains the prompt and assertions. The agent never sees this file.

## Writing Evals

Create a folder with your starting code and add `__eval__.ts`:

```ts
// evals/add-button/__eval__.ts
export const prompt = "Add a logout button to the header"
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
| `prompt` | `string` | What to ask the agent |
| `scripts` | `string[]` | npm scripts that must exit 0 (optional) |

### The `files` Object

The framework provides `files`—a `Record<string, string>` mapping file paths to contents. This is populated with the agent's output (excluding `node_modules`, `.git`, lockfiles, etc.) before tests run.

```ts
import { files } from '@vercel/eval-framework'

files['src/App.tsx']              // get specific file
Object.keys(files)                // list all paths
Object.values(files)              // all contents
```

Use standard vitest assertions. The framework doesn't abstract vitest—you get full access to all its features.

## Execution Flow

1. Copy eval folder to sandbox (excluding `__eval__.ts`)
2. Run `npm install`
3. Run agent with `prompt`
4. Populate `files` with output
5. Inject `__eval__.ts`
6. Run vitest
7. Run `scripts` (if defined)
8. Result: **0 or 1**

## Scoring

An eval is binary: **pass (1) or fail (0)**. No partial scores.

For an eval to pass:
- All vitest tests must pass
- All `scripts` must exit 0

## Defining Agents

An agent is just a function. Define it anywhere you want:

```ts
// my-agent.ts
import type { Agent } from '@vercel/eval-framework'

export const claude: Agent = async (prompt, sandbox) => {
  await sandbox.exec('npm i -g @anthropic-ai/claude-code')
  await sandbox.exec(`claude --print -p "${prompt}"`)

  const transcript = await sandbox.readFile('~/.claude/.../transcript.jsonl')
  return JSON.parse(transcript)
}
```

The transcript is stored for later investigation. When an eval fails, analyze it to understand what the agent did and where it went wrong.

## CLI

```bash
# Single run
npx eval my-agent.ts

# Run specific eval
npx eval my-agent.ts add-button

# Best of N (stop on first pass)
npx eval my-agent.ts --best-of 5

# Flakiness testing (run all, report stats)
npx eval my-agent.ts --runs 10

# Compare agents
npx eval agent-a.ts agent-b.ts --runs 10
```

## CLI Output

### Single Run

```
npx eval my-agent.ts add-button

add-button ✓ PASS (45.2s)
```

### Flakiness Report

```
npx eval my-agent.ts --runs 10

add-button (10 runs)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Pass rate:  7/10 (70%)
Mean:       45.2s
Stddev:     8.3s
```

### Agent Comparison

```
npx eval agent-a.ts agent-b.ts --runs 10

add-button (10 runs each)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
            agent-a     agent-b
Pass rate   4/10        9/10
Mean        48.2s       41.3s

Winner: agent-b (90%)
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
    "agent": "my-agent.ts",
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
