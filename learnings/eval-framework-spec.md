# @vercel/eval-framework - Technical Specification

**Author:** judegao
**Date:** 2026-01-26

## Overview

`@vercel/eval-framework` is an opinionated evaluation framework for testing AI coding agents on Node.js projects.

**Assumptions:** npm, vitest, claude-code CLI, Vercel Sandbox

## Project Structure

```
my-evals/
├── evals/
│   └── add-button/           # Eval fixture (normal Node.js project)
│       ├── src/
│       │   └── App.tsx
│       ├── package.json
│       ├── PROMPT.md         # Prompt sent to agent
│       └── EVAL.ts           # Validation tests (hidden from agent)
├── experiments/
│   └── with-skill.ts
├── .env                      # API keys (auto-loaded)
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

Validation tests. The framework provides a global `sandbox` object at runtime:

```ts
// evals/add-button/EVAL.ts
import { sandbox } from '@vercel/eval-framework'
import { test, expect } from 'vitest'

test('logout button exists somewhere in codebase', async () => {
  const files = await sandbox.glob('**/*.tsx')
  for (const file of files) {
    const content = await sandbox.readFile(file)
    if (/logout/i.test(content)) {
      expect(true).toBe(true)
      return
    }
  }
  expect(false).toBe(true)
})

test('app builds successfully', async () => {
  const result = await sandbox.exec('npm run build')
  expect(result.exitCode).toBe(0)
})
```

**Import resolution:** The `@vercel/eval-framework` import is resolved by the framework's test harness at runtime, not by the fixture's node_modules. The framework injects `EVAL.ts` into the sandbox after the agent runs and provides the `sandbox` global when vitest executes.

## Execution Flow

1. Copy eval folder to sandbox (excluding `PROMPT.md`, `EVAL.ts`)
2. Run `npm install`
3. Run `setup` (if defined in experiment). **If setup fails, eval fails.**
4. Run agent with contents of `PROMPT.md` (passed as CLI argument to claude-code)
5. Inject `EVAL.ts` into project root
6. Run `scripts` in order (if defined). **Stops on first failure.** (runs `npm run <script>`)
7. Run `npx vitest run EVAL.ts`
8. Result: **0 or 1**
9. Pretty-print summary to terminal with path to detailed results

**Concurrency:** All evals and runs execute concurrently in Vercel Sandbox.

## Scoring

An eval is binary: **pass (1) or fail (0)**. No partial scores.

For an eval to pass:
- Setup must succeed (if defined)
- All `scripts` must exit 0
- All vitest tests must pass

## CLI

```bash
npx eval <experiment>
```

Experiment file defines everything.

```bash
npx eval experiments/with-skill.ts
npx eval experiments/baseline.ts
```

### Scaffolding

Create a new eval project with example files:

```bash
npx eval init
```

Creates:

```
my-evals/
├── evals/
│   └── hello-world/
│       ├── src/
│       │   └── index.ts
│       ├── package.json
│       ├── PROMPT.md
│       └── EVAL.ts
├── experiments/
│   └── default.ts
├── .env.example
└── package.json
```

## Experiment Config

```ts
// experiments/with-skill.ts
export default {
  agent: 'claude-code',
  model: 'opus',
  scripts: ['build', 'lint'],
  setup: async (sandbox) => {
    await sandbox.exec('chmod +x ./skills.sh && ./skills.sh')
  },
  runs: 10,
  evals: ['add-button', 'fix-auth'],
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `agent` | `'claude-code'` | `'claude-code'` | Which agent CLI to use |
| `model` | `string` | `'sonnet'` | Model to use (e.g., `'opus'`, `'sonnet'`, `'haiku'`) |
| `scripts` | `string[]` | `[]` | npm scripts that must exit 0 (runs `npm run <script>`) |
| `setup` | `(sandbox) => Promise<void>` | none | Setup function that runs before agent starts |
| `runs` | `number` | `1` | Number of runs per eval |
| `earlyExit` | `boolean` | `true` | Stop after first passing run (set `false` to always run all) |
| `evals` | `string \| string[] \| (name: string) => boolean` | all | Which evals to run |

### The `evals` Field

```ts
// Run all evals (default)
evals: undefined

// Run single eval by exact name
evals: 'add-button'

// Run multiple evals by name
evals: ['add-button', 'fix-auth']

// Run evals matching a predicate
evals: (name) => name.startsWith('auth-')
```

### Minimal Config

```ts
// experiments/baseline.ts
export default {}
```

Uses all defaults: claude-code agent, sonnet model, 1 run, all evals.

### Full Config (with comments)

```ts
// experiments/example.ts
export default {
  // AGENT (primary choice)
  // Which agent CLI to use for running evals
  // Options: 'claude-code'
  // Default: 'claude-code'
  agent: 'claude-code',

  // MODEL (secondary to agent)
  // Model to pass to the agent
  // Default: 'sonnet'
  model: 'opus',

  // SCRIPTS
  // npm scripts to run after agent completes, before tests
  // Runs `npm run <script>` for each
  // Stops on first failure
  // Default: []
  scripts: ['build', 'lint'],

  // SETUP
  // Setup function that runs before agent starts
  // Receives sandbox instance for file/command operations
  // If setup throws, eval fails immediately
  // Default: none
  setup: async (sandbox) => {
    await sandbox.exec('chmod +x ./skills.sh && ./skills.sh')
  },

  // RUNS
  // Number of times to run each eval
  // Results aggregated in summary.json
  // Default: 1
  runs: 10,

  // EARLY EXIT
  // Stop running after first pass
  // Set to false to always run all attempts
  // Default: true
  earlyExit: true,

  // EVALS
  // Which evals to run
  // - undefined: all evals in evals/
  // - string: exact match
  // - string[]: list of exact matches
  // - (name) => boolean: predicate function
  // Default: all
  evals: ['add-button', 'fix-auth'],
}
```

## Results

Each experiment run produces results in `results/<experiment-name>/<timestamp>/`:

- **Experiment name:** Derived from filename without extension (e.g., `with-skill.ts` → `with-skill`)
- **Timestamp:** ISO8601 format, UTC, colons replaced with dashes (e.g., `2026-01-26T12-00-00Z`)

```
results/with-skill/2026-01-26T12-00-00Z/
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

### Terminal Output

After each eval completes, pretty-print summary to terminal:

```
┌─────────────────────────────────────────────────┐
│  add-button                                     │
│  ✓ 7/10 passed (70%)                            │
│  Mean duration: 45.2s                           │
│                                                 │
│  Details: results/with-skill/2026-01-26T12.../  │
└─────────────────────────────────────────────────┘
```

### result.json

Small, scannable. Pointers to large files:

```json
{
  "eval": "add-button",
  "run": 1,
  "passed": false,
  "duration": 45200,
  "setup": { "passed": true, "duration": 1200 },
  "scripts": {
    "build": { "passed": true, "duration": 12300, "output": "./outputs/build.txt" },
    "lint": { "passed": false, "duration": 2100, "output": "./outputs/lint.txt" }
  },
  "tests": {
    "passed": false,
    "failures": ["logout button exists"],
    "output": "./outputs/tests.txt"
  },
  "transcript": "./transcript.jsonl",
  "config": {
    "agent": "claude-code",
    "model": "opus"
  },
  "timestamp": "2026-01-26T12:00:00Z"
}
```

### summary.json

Aggregated stats across runs (always generated):

```json
{
  "eval": "add-button",
  "runs": 10,
  "passed": 7,
  "passRate": 0.7,
  "meanDuration": 45200,
  "stddev": 8300,
  "earlyExit": true,
  "stoppedEarly": true,
  "attemptsUntilPass": 3
}
```

### transcript.jsonl

Full agent output. Format depends on agent (JSONL for claude-code). Framework stores it as-is, doesn't parse it. Use this for debugging failed evals.

## Reporting

The framework outputs structured JSON. Users build their own reports, dashboards, or analysis scripts from the results.

## Sandbox

Vercel Sandbox only. No local sandbox support.

```ts
interface Sandbox {
  exec(cmd: string): Promise<{ stdout: string; stderr: string; exitCode: number }>
  readFile(path: string): Promise<string>
  writeFile(path: string, content: string): Promise<void>
  glob(pattern?: string): Promise<string[]>  // default: '**/*'
}
```

Same interface used in both `setup` config and EVAL.ts.

### Vercel Sandbox

Firecracker MicroVMs via `@vercel/sandbox`. Hard isolation, ephemeral, auto-cleanup.

- Runtime: Node.js 24 (Amazon Linux 2023)
- Writable directory: `/vercel/sandbox`
- Pre-installed: git, npm, pnpm, common build tools
- Requires: `VERCEL_TOKEN` or `VERCEL_OIDC_TOKEN` environment variable

### Why No Local Sandbox?

1. **Security** — Agents execute arbitrary commands; isolation prevents damage
2. **Permissions** — Skipping permission checks is dangerous and varies by OS
3. **Reproducibility** — Local environment differences cause inconsistent results

## Environment Variables

Loaded automatically from `.env`:

```bash
# .env

# Vercel Sandbox (required)
# Use one of these:
VERCEL_TOKEN=
VERCEL_OIDC_TOKEN=

# Claude Code agent (required)
ANTHROPIC_API_KEY=
```
