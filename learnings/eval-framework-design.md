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

import { files, exec } from '@vercel/eval-framework'
import { test, expect } from 'vitest'

test('logout button exists somewhere in codebase', () => {
  const hasLogout = Object.values(files).some(c => /logout/i.test(c))
  expect(hasLogout).toBe(true)
})

test('logout button has click handler', () => {
  const hasHandler = Object.values(files).some(c => /onClick.*logout/i.test(c))
  expect(hasHandler).toBe(true)
})

test('app starts without errors', async () => {
  const result = await exec('npm run dev -- --port 3333 &')
  await exec('sleep 2')
  const health = await exec('curl -s http://localhost:3333')
  expect(health.exitCode).toBe(0)
})
```

### Exports

| Export | Type | Description |
|--------|------|-------------|
| `scripts` | `string[]` | npm scripts that must exit 0 (optional, default: `[]`) |

### The `files` Object

The framework provides `files`—a `Record<string, string>` mapping file paths to contents.

**Paths:** Relative to project root, no leading `./` (e.g., `src/App.tsx`, not `./src/App.tsx`)

**Excluded:** `node_modules/`, `.git/`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `EVAL.ts`

**Binary files:** Excluded. Only text files are included.

```ts
import { files } from '@vercel/eval-framework'

files['src/App.tsx']              // get specific file
Object.keys(files)                // list all paths
Object.values(files)              // all contents
```

### The `exec` Function

The framework provides `exec`—runs commands in the same sandbox as the eval. Useful for probing runtime behavior.

```ts
import { exec } from '@vercel/eval-framework'

const result = await exec('npm run build')
// result: { stdout: string, stderr: string, exitCode: number }

expect(result.exitCode).toBe(0)
expect(result.stdout).toContain('Build completed')
```

Use standard vitest assertions. The framework doesn't abstract vitest—you get full access to all its features.

## Execution Flow

1. Copy eval folder to sandbox (excluding `PROMPT.md`, `EVAL.ts`)
2. Run `npm install`
3. Run prehook (if defined in config). **If prehook fails, eval fails.**
4. Run agent with contents of `PROMPT.md` (passed as CLI argument to claude-code)
5. Populate `files` with output
6. Inject `EVAL.ts` into project root
7. Run `scripts` in order (if defined). **Stops on first failure.**
8. Run `npx vitest run EVAL.ts`
9. Result: **0 or 1**

**Timeouts:** Agent has 10 minute timeout. Scripts have 2 minute timeout each.

**Parallelism:** Evals run sequentially. Multiple runs of the same eval run sequentially.

**Sandbox cleanup:** Sandbox is deleted after each run. For debugging, use `sandbox: 'local'` in config.

## Scoring

An eval is binary: **pass (1) or fail (0)**. No partial scores.

For an eval to pass:
- Prehook must succeed (if defined)
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
  prehook: async (sandbox) => {
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
| `sandbox` | `'vercel' \| 'local' \| 'docker'` | `'vercel'` | Sandbox provider for isolated execution |
| `prehook` | `(sandbox) => Promise<void>` | none | Setup before agent runs |
| `runs` | `number` | `1` | Number of runs per eval |
| `bestOf` | `number` | none | Stop on first pass (mutually exclusive with `runs`) |
| `evals` | `string[]` | all | Which evals to run (folder names) |

**`runs` vs `bestOf`:** Cannot specify both. If both specified, error.

### Minimal Config

```ts
// configs/baseline.ts
export default {}
```

Uses all defaults: claude-code agent, sonnet model, vercel sandbox, 1 run, all evals.

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

  // Sandbox provider for isolated execution
  // Options: 'vercel' | 'local' | 'docker'
  // Default: 'vercel'
  // - 'vercel': Firecracker MicroVMs via @vercel/sandbox (recommended)
  // - 'local': Local filesystem (no isolation, use for debugging)
  // - 'docker': Docker containers (requires Docker daemon)
  sandbox: 'vercel',

  // Setup function that runs before agent starts
  // Receives sandbox instance for file/command operations
  // If prehook throws, eval fails immediately
  // Default: none
  prehook: async (sandbox) => {
    await sandbox.exec('npx @vercel/next-skill install')
  },

  // Number of times to run each eval
  // Results aggregated in summary.json
  // Default: 1
  // Cannot be used with 'bestOf'
  runs: 10,

  // Stop after first passing run
  // Useful for "can this ever pass?" testing
  // Cannot be used with 'runs'
  // bestOf: 5,

  // Which evals to run (folder names in evals/)
  // Default: all evals found in evals/
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
  "prehook": { "passed": true, "duration": 1200 },
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

Aggregated stats across runs (only generated when `runs > 1` or `bestOf` is set):

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

For `bestOf`, also includes:

```json
{
  "eval": "add-button",
  "bestOf": 5,
  "passed": true,
  "attemptsUntilPass": 3
}
```

### transcript.jsonl

Full agent output. Format depends on agent (JSONL for claude-code). Framework stores it as-is, doesn't parse it.

## Reporting

The framework outputs structured JSON. Users build their own reports, dashboards, or analysis scripts from the results.

## Sandbox

Configured via the `sandbox` field in config. Default: `'vercel'`.

```ts
interface Sandbox {
  exec(cmd: string): Promise<{ stdout: string; stderr: string; exitCode: number }>
  readFile(path: string): Promise<string>
  writeFile(path: string, content: string): Promise<void>
}
```

### Providers

**`'vercel'` (default):** Firecracker MicroVMs via `@vercel/sandbox`. Hard isolation, ephemeral, auto-cleanup. Requires Vercel authentication (OIDC or access token).

- Runtime: Node.js 24 (Amazon Linux 2023)
- Writable directory: `/vercel/sandbox`
- Pre-installed: git, npm, pnpm, common build tools
- Max timeout: 5 hours (Pro/Enterprise), 45 minutes (Hobby)

**`'local'`:** Local filesystem. No isolation—runs in temp directory on host machine. Useful for debugging.

**`'docker'`:** Docker containers. Requires Docker daemon running locally.
