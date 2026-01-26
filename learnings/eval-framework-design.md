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
├── configs/
│   └── with-skill.ts
└── results/
```

The eval folder **is** the fixture—a normal Node.js project. Two special files:
- `PROMPT.md` — the prompt sent to the agent (static markdown)
- `EVAL.ts` — validation tests (agent never sees this, can import from fixture)

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
3. Run prehook (if defined in config)
4. Run agent with contents of `PROMPT.md`
5. Populate `files` with output
6. Inject `EVAL.ts`
7. Run vitest
8. Run `scripts` (if defined)
9. Result: **0 or 1**

## Scoring

An eval is binary: **pass (1) or fail (0)**. No partial scores.

For an eval to pass:
- All vitest tests must pass
- All `scripts` must exit 0

## CLI

```bash
npx eval <config>
```

That's it. Config file defines everything.

```bash
npx eval configs/with-skill.ts
npx eval configs/baseline.ts
```

## Configs

```ts
// configs/with-skill.ts
export default {
  agent: 'claude-code',           // enum: 'claude-code' | future agents
  model: 'opus',
  prehook: async (sandbox) => {
    await sandbox.exec('npx @vercel/next-skill install')
  },
  runs: 10,
  evals: ['add-button', 'fix-auth'],  // or '*' for all
}
```

| Field | Type | Description |
|-------|------|-------------|
| `agent` | `'claude-code'` | Which agent to use (default: `'claude-code'`) |
| `model` | `string` | Model to use (e.g., `'opus'`, `'sonnet'`) |
| `prehook` | `(sandbox) => Promise<void>` | Setup before agent runs (optional) |
| `runs` | `number` | Number of runs per eval (default: 1) |
| `bestOf` | `number` | Stop on first pass (alternative to `runs`) |
| `evals` | `string[]` | Which evals to run (default: all) |

Minimal config:

```ts
// configs/baseline.ts
export default {
  model: 'sonnet',
}
```

## Results

Each config run produces results in `results/<config-name>/<timestamp>/`:

```
results/with-skill/2026-01-26T12-00-00/
├── add-button/
│   ├── run-1/
│   │   ├── result.json
│   │   ├── transcript.jsonl
│   │   └── outputs/
│   │       ├── build.txt
│   │       ├── lint.txt
│   │       └── tests.txt
│   ├── run-2/
│   │   └── ...
│   └── summary.json
└── fix-auth/
    └── ...
```

### result.json

Small, scannable. Pointers to large files:

```json
{
  "eval": "add-button",
  "passed": false,
  "duration": 45200,
  "scripts": {
    "build": { "passed": true, "duration": 12300, "output": "./outputs/build.txt" },
    "lint": { "passed": false, "duration": 2100, "output": "./outputs/lint.txt" }
  },
  "tests": {
    "passed": false,
    "failures": ["logout button exists"],
    "output": "./outputs/tests.txt"
  },
  "transcript": "./transcript.jsonl"
}
```

### summary.json

Aggregated stats across runs:

```json
{
  "eval": "add-button",
  "runs": 10,
  "passed": 7,
  "passRate": 0.7,
  "meanDuration": 45200,
  "stddev": 8300
}
```

### transcript.jsonl

Full agent output. Format depends on agent (JSONL for claude-code). Framework stores it as-is, doesn't parse it.

## Reporting

The framework outputs structured JSON. Users build their own reports, dashboards, or analysis scripts from the results.

## Sandbox

```ts
interface Sandbox {
  exec(cmd: string): Promise<{ stdout: string; stderr: string; exitCode: number }>
  readFile(path: string): Promise<string>
  writeFile(path: string, content: string): Promise<void>
}
```

Providers: local, Docker, Vercel Sandbox.
