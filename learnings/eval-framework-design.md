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

Validation tests. The framework provides a global `sandbox` object:

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

test('app starts without errors', async () => {
  await sandbox.exec('npm run dev -- --port 3333 &')
  await sandbox.exec('sleep 2')
  const health = await sandbox.exec('curl -s http://localhost:3333')
  expect(health.exitCode).toBe(0)
})
```

Use standard vitest assertions. The framework doesn't abstract vitest—you get full access to all its features.

## Execution Flow

1. Copy eval folder to sandbox (excluding `PROMPT.md`, `EVAL.ts`)
2. Run `npm install`
3. Run `setup` (if defined in config). **If setup fails, eval fails.**
4. Run agent with contents of `PROMPT.md` (passed as CLI argument to claude-code)
5. Inject `EVAL.ts` into project root
6. Run `scripts` in order (if defined in config). **Stops on first failure.**
7. Run `npx vitest run EVAL.ts`
8. Result: **0 or 1**

**Concurrency:** All evals and runs execute concurrently (in vercel sandbox mode).

## Scoring

An eval is binary: **pass (1) or fail (0)**. No partial scores.

For an eval to pass:
- Setup must succeed (if defined)
- All `scripts` must exit 0
- All vitest tests must pass

## CLI

```bash
npx eval <config>
```

That's it. Config file defines everything.

```bash
npx eval configs/with-skill.ts
npx eval configs/baseline.ts
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
├── configs/
│   └── default.ts
└── package.json
```

## Configs

```ts
// configs/with-skill.ts
export default {
  agent: 'claude-code',
  model: 'opus',
  scripts: ['build', 'lint'],
  setup: async (sandbox) => {
    await sandbox.exec('npx @vercel/next-skill install')
  },
  runs: 10,
  evals: ['add-button', 'fix-auth'],
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `agent` | `'claude-code'` | `'claude-code'` | Which agent to use |
| `model` | `string` | `'sonnet'` | Model to use (e.g., `'opus'`, `'sonnet'`) |
| `env` | `Record<string, string>` | `{}` | Environment variables to pass to the agent |
| `scripts` | `string[]` | `[]` | npm scripts that must exit 0 before tests run |
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
// configs/baseline.ts
export default {}
```

Uses all defaults: claude-code agent, sonnet model, 1 run, all evals.

### Full Config (with comments)

```ts
// configs/example.ts
export default {
  // Which agent to use for running evals
  // Options: 'claude-code'
  // Default: 'claude-code'
  agent: 'claude-code',

  // Model to pass to the agent
  // Default: 'sonnet'
  model: 'opus',

  // Environment variables passed to the agent
  // Required vars depend on agent:
  // - claude-code: ANTHROPIC_API_KEY (required)
  // Default: {}
  env: {
    ANTHROPIC_API_KEY: process.env.ANTHROPIC_API_KEY,
  },

  // npm scripts to run after agent completes, before tests
  // Stops on first failure
  // Default: []
  scripts: ['build', 'lint'],

  // Setup function that runs before agent starts
  // Receives sandbox instance for file/command operations
  // If setup throws, eval fails immediately
  // Default: none
  setup: async (sandbox) => {
    await sandbox.exec('npx @vercel/next-skill install')
  },

  // Number of times to run each eval
  // Results aggregated in summary.json
  // Default: 1
  runs: 10,

  // Stop running after first pass
  // Set to false to always run all attempts
  // Default: true
  earlyExit: true,

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

Each config run produces results in `results/<config-name>/<timestamp>/`:

- **Config name:** Derived from filename without extension (e.g., `with-skill.ts` → `with-skill`)
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

The framework auto-detects sandbox provider based on available credentials:

1. **Vercel token available** → uses `@vercel/sandbox` (logged: `Using vercel sandbox`)
2. **No token** → uses local sandbox (logged: `Using local sandbox`)

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
- Requires: `VERCEL_TOKEN` environment variable

### Local Sandbox

Local filesystem. No isolation—runs in temp directory on host machine.

## Environment Variables

```bash
# .env.example

# Vercel Sandbox (optional)
# If set, uses Vercel's isolated MicroVMs
# If not set, falls back to local sandbox
VERCEL_TOKEN=

# Claude Code agent (required)
# API key for Anthropic's Claude API
ANTHROPIC_API_KEY=
```
