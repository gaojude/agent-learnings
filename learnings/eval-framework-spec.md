# @vercel/eval-framework — Engineering Design Document

**Author:** judegao
**Date:** 2026-01-26
**Status:** Draft
**Version:** 1.0

---

## Table of Contents

1. [Overview](#overview)
2. [Goals and Non-Goals](#goals-and-non-goals)
3. [Architecture](#architecture)
4. [Core Components](#core-components)
5. [Data Flow](#data-flow)
6. [Experiment Configuration](#experiment-configuration)
7. [Eval Fixtures](#eval-fixtures)
8. [Sandbox Environment](#sandbox-environment)
9. [Agent Execution](#agent-execution)
10. [Test Execution](#test-execution)
11. [Scoring System](#scoring-system)
12. [Results and Reporting](#results-and-reporting)
13. [CLI Interface](#cli-interface)
14. [Error Handling](#error-handling)
15. [Concurrency Model](#concurrency-model)
16. [Security Considerations](#security-considerations)
17. [Environment Variables](#environment-variables)
18. [Future Considerations](#future-considerations)

---

## Overview

`@vercel/eval-framework` is an opinionated evaluation framework for testing AI coding agents on Node.js projects. It provides a structured way to measure whether an AI agent can successfully complete coding tasks by:

1. Presenting the agent with a task (PROMPT.md)
2. Running the agent in an isolated sandbox environment
3. Validating the agent's work using vitest tests (EVAL.ts)
4. Aggregating results across multiple runs to measure reliability

### Core Assumptions

- **Package Manager:** npm (pnpm support planned)
- **Test Framework:** vitest (no abstraction layer—full vitest access)
- **Agent CLI:** claude-code (extensible to other agents)
- **Execution Environment:** Vercel Sandbox (Firecracker MicroVMs)
- **Project Type:** Node.js projects with package.json

---

## Goals and Non-Goals

### Goals

1. **Reproducible evaluations** — Same eval, same agent, same results distribution
2. **Isolated execution** — Agent cannot affect host system or other evals
3. **Simple authoring** — Evals are just Node.js projects with two special files
4. **Statistical rigor** — Support multiple runs to measure pass rates
5. **Debuggability** — Full transcripts and logs for failed runs
6. **Extensibility** — Support for different agents, models, and configurations

### Non-Goals

1. **Partial scoring** — Evals are binary pass/fail only
2. **Local execution** — No support for running without sandbox isolation
3. **Non-Node.js projects** — Python, Go, etc. are out of scope
4. **Custom test frameworks** — vitest only, no Jest/Mocha support
5. **Real-time monitoring** — Results are batch-processed after completion

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLI (npx eval)                          │
├─────────────────────────────────────────────────────────────────┤
│                      Experiment Runner                          │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │  Config   │  │   Eval    │  │  Sandbox  │  │  Results  │    │
│  │  Loader   │  │  Scanner  │  │  Manager  │  │  Writer   │    │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘    │
├─────────────────────────────────────────────────────────────────┤
│                      Eval Executor (per eval)                   │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │   Setup   │  │   Agent   │  │  Scripts  │  │   Test    │    │
│  │  Runner   │  │  Runner   │  │  Runner   │  │  Runner   │    │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘    │
├─────────────────────────────────────────────────────────────────┤
│                      Vercel Sandbox                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Firecracker MicroVM                         │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐     │   │
│  │  │ Node.js │  │   npm   │  │   git   │  │  vitest │     │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘     │   │
│  │                                                          │   │
│  │  /vercel/sandbox/                                        │   │
│  │  ├── src/                                                │   │
│  │  ├── package.json                                        │   │
│  │  └── EVAL.ts (injected after agent)                      │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Module Dependency Graph

```
cli.ts
  └── ExperimentRunner
        ├── ConfigLoader
        │     └── validates experiment config schema
        ├── EvalScanner
        │     └── discovers evals in evals/ directory
        ├── SandboxManager
        │     └── @vercel/sandbox (Firecracker API)
        ├── EvalExecutor
        │     ├── SetupRunner
        │     ├── AgentRunner
        │     │     └── spawns claude-code CLI
        │     ├── ScriptRunner
        │     │     └── runs npm scripts
        │     └── TestRunner
        │           └── runs vitest
        └── ResultsWriter
              └── writes JSON to results/
```

---

## Core Components

### 1. ConfigLoader

**Responsibility:** Load and validate experiment configuration files.

**Input:** Path to experiment file (e.g., `experiments/my-experiment.ts`)

**Output:** Validated `ExperimentConfig` object

**Behavior:**
- Dynamically imports the TypeScript/JavaScript config file
- Validates against the `ExperimentConfig` schema
- Applies defaults for missing fields
- Throws descriptive errors for invalid configs

```ts
interface ExperimentConfig {
  agent: 'claude-code'
  model: string                                    // default: 'opus'
  evals: string | string[] | ((name: string) => boolean) | undefined  // default: all
  runs: number                                     // default: 1
  earlyExit: boolean                               // default: true
  scripts: string[]                                // default: []
  setup?: (sandbox: Sandbox) => Promise<void>
}
```

**Default Resolution:**

| Field | Default Value | Rationale |
|-------|---------------|-----------|
| `agent` | `'claude-code'` | Only supported agent currently |
| `model` | `'opus'` | Best performance for complex coding tasks |
| `evals` | `undefined` (all) | Run everything by default |
| `runs` | `1` | Single run for quick iteration |
| `earlyExit` | `true` | Stop early to save time/cost |
| `scripts` | `[]` | No scripts required by default |
| `setup` | `undefined` | No setup required by default |

### 2. EvalScanner

**Responsibility:** Discover eval fixtures in the `evals/` directory.

**Input:** Path to evals directory, optional filter from config

**Output:** List of `EvalFixture` objects

**Behavior:**
- Scans `evals/` for directories containing both `PROMPT.md` and `EVAL.ts`
- Filters based on `config.evals` (string match, array, or predicate)
- Validates that each fixture has a valid `package.json`
- Returns fixtures sorted alphabetically by name

```ts
interface EvalFixture {
  name: string           // directory name (e.g., 'add-button')
  path: string           // absolute path to fixture directory
  promptPath: string     // path to PROMPT.md
  evalPath: string       // path to EVAL.ts
  packageJson: object    // parsed package.json
}
```

**Error Conditions:**
- No `evals/` directory found → error with instructions
- Eval directory missing `PROMPT.md` → warning, skip eval
- Eval directory missing `EVAL.ts` → warning, skip eval
- Eval directory missing `package.json` → error, fixture invalid

### 3. SandboxManager

**Responsibility:** Create, configure, and manage Vercel Sandbox instances.

**Input:** Eval fixture, experiment config

**Output:** Configured `Sandbox` instance

**Behavior:**
- Creates a new Firecracker MicroVM via `@vercel/sandbox`
- Copies fixture files to `/vercel/sandbox/` (excluding `PROMPT.md`, `EVAL.ts`)
- Runs `npm install` to install dependencies
- Returns sandbox interface for subsequent operations

```ts
interface Sandbox {
  // Execute a shell command
  exec(cmd: string): Promise<{
    stdout: string
    stderr: string
    exitCode: number
  }>

  // Read a file from the sandbox
  readFile(path: string): Promise<string>

  // Write a file to the sandbox
  writeFile(path: string, content: string): Promise<void>

  // Find files matching a glob pattern
  glob(pattern?: string): Promise<string[]>  // default: '**/*'

  // Cleanup and destroy the sandbox
  destroy(): Promise<void>
}
```

**Sandbox Lifecycle:**

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Create  │ ──► │  Copy   │ ──► │ Install │ ──► │  Ready  │
│   VM    │     │ Files   │     │   npm   │     │         │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
                                                     │
                                                     ▼
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Destroy │ ◄── │ Collect │ ◄── │  Test   │ ◄── │  Agent  │
│   VM    │     │ Results │     │  Phase  │     │  Phase  │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
```

### 4. SetupRunner

**Responsibility:** Execute the optional `setup` function before the agent runs.

**Input:** Sandbox instance, setup function from config

**Output:** Success/failure status with duration

**Behavior:**
- If no setup function defined, skip and return success
- Execute setup function with sandbox instance
- Catch any errors and mark eval as failed
- Record duration for reporting

```ts
interface SetupResult {
  passed: boolean
  duration: number      // milliseconds
  error?: string        // error message if failed
}
```

**Error Handling:**
- Setup function throws → eval fails immediately, no agent execution
- Setup function times out (5 min default) → eval fails
- Setup function returns normally → continue to agent execution

### 5. AgentRunner

**Responsibility:** Execute the AI agent with the task prompt.

**Input:** Sandbox instance, prompt content, agent config

**Output:** Agent transcript and completion status

**Behavior:**
- Read `PROMPT.md` content from fixture
- Spawn agent CLI process inside sandbox
- Pass prompt as CLI argument or stdin
- Capture full transcript (stdout/stderr)
- Wait for agent to complete or timeout

```ts
interface AgentResult {
  completed: boolean
  duration: number           // milliseconds
  transcript: string         // full agent output (JSONL for claude-code)
  exitCode: number
}
```

**Agent CLI Invocation:**

```bash
# For claude-code agent
claude-code --model opus --print --output-format stream-json <<< "$PROMPT"
```

**Agent-Specific Configurations:**

| Agent | CLI | Model Flag | Output Format |
|-------|-----|------------|---------------|
| `claude-code` | `claude-code` | `--model` | `--output-format stream-json` |

**Timeout Handling:**
- Default timeout: 10 minutes per agent run
- On timeout: kill process, mark as failed, continue to results

### 6. ScriptRunner

**Responsibility:** Execute npm scripts defined in the experiment config.

**Input:** Sandbox instance, list of script names

**Output:** Per-script results with outputs

**Behavior:**
- Run scripts in order: `npm run <script>`
- Stop on first failure (non-zero exit code)
- Capture stdout/stderr for each script
- Record duration for each script

```ts
interface ScriptResult {
  name: string
  passed: boolean
  duration: number
  exitCode: number
  output: string         // combined stdout + stderr
}

interface ScriptsResult {
  passed: boolean        // all scripts passed
  scripts: ScriptResult[]
  stoppedAt?: string     // name of first failed script
}
```

**Example Scripts:**
- `build` → `npm run build` (compile TypeScript, bundle, etc.)
- `lint` → `npm run lint` (ESLint, Prettier, etc.)
- `typecheck` → `npm run typecheck` (tsc --noEmit)

### 7. TestRunner

**Responsibility:** Execute EVAL.ts validation tests using vitest.

**Input:** Sandbox instance, path to EVAL.ts

**Output:** Test results with pass/fail status

**Behavior:**
- Inject `EVAL.ts` into sandbox root directory
- Configure vitest to use framework's test harness
- Run `npx vitest run EVAL.ts`
- Parse vitest JSON output for results
- Extract failed test names for reporting

```ts
interface TestResult {
  passed: boolean
  total: number          // total test count
  passed_count: number   // passed test count
  failed_count: number   // failed test count
  failures: string[]     // names of failed tests
  duration: number
  output: string         // full vitest output
}
```

**EVAL.ts Injection Process:**

1. Read `EVAL.ts` from fixture (excluded during initial copy)
2. Write to `/vercel/sandbox/EVAL.ts`
3. Inject sandbox global via vitest setup file
4. Run vitest with framework's configuration

**Vitest Configuration (injected):**

```ts
// vitest.config.ts (injected by framework)
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals: true,
    setupFiles: ['./eval-framework-setup.ts'],
    include: ['EVAL.ts'],
  },
})
```

**Setup File (injected):**

```ts
// eval-framework-setup.ts (injected by framework)
import { createSandboxClient } from '@vercel/eval-framework/runtime'

// Make sandbox available globally
globalThis.sandbox = createSandboxClient()
```

### 8. ResultsWriter

**Responsibility:** Write evaluation results to the filesystem.

**Input:** Evaluation results, experiment config

**Output:** Result files in `results/` directory

**Behavior:**
- Create directory structure for results
- Write individual run results (`result.json`)
- Write agent transcripts (`transcript.jsonl`)
- Write script/test outputs to `outputs/`
- Generate summary statistics (`summary.json`)
- Pretty-print summary to terminal

---

## Data Flow

### Complete Execution Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. CLI INVOCATION                                               │
│    npx eval experiments/my-experiment.ts                        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. CONFIG LOADING                                               │
│    - Load experiments/my-experiment.ts                          │
│    - Validate against schema                                    │
│    - Apply defaults                                             │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. EVAL DISCOVERY                                               │
│    - Scan evals/ directory                                      │
│    - Filter based on config.evals                               │
│    - Validate fixtures (PROMPT.md, EVAL.ts, package.json)       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. PARALLEL EXECUTION (for each eval × runs)                    │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ 4a. SANDBOX CREATION                                    │  │
│    │     - Create Firecracker MicroVM                        │  │
│    │     - Copy fixture files (exclude PROMPT.md, EVAL.ts)   │  │
│    │     - Run npm install                                   │  │
│    └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ 4b. SETUP PHASE                                         │  │
│    │     - Run config.setup(sandbox) if defined              │  │
│    │     - If setup fails → eval fails, skip remaining       │  │
│    └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ 4c. AGENT PHASE                                         │  │
│    │     - Read PROMPT.md content                            │  │
│    │     - Spawn claude-code with prompt                     │  │
│    │     - Capture transcript                                │  │
│    │     - Wait for completion                               │  │
│    └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ 4d. EVAL.ts INJECTION                                   │  │
│    │     - Copy EVAL.ts to sandbox root                      │  │
│    │     - Inject vitest config                              │  │
│    │     - Inject sandbox global setup                       │  │
│    └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ 4e. SCRIPTS PHASE                                       │  │
│    │     - Run each script in order: npm run <script>        │  │
│    │     - Stop on first failure                             │  │
│    │     - Record outputs                                    │  │
│    └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ 4f. TEST PHASE                                          │  │
│    │     - Run: npx vitest run EVAL.ts                       │  │
│    │     - Parse results                                     │  │
│    │     - Record failures                                   │  │
│    └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│                              ▼                                   │
│    ┌─────────────────────────────────────────────────────────┐  │
│    │ 4g. CLEANUP                                             │  │
│    │     - Destroy sandbox                                   │  │
│    │     - Free resources                                    │  │
│    └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. RESULTS AGGREGATION                                          │
│    - Collect results from all runs                              │
│    - Calculate pass rates, mean duration, stddev                │
│    - Handle early exit if configured                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. OUTPUT                                                       │
│    - Write result.json for each run                             │
│    - Write summary.json for each eval                           │
│    - Write transcript.jsonl                                     │
│    - Write outputs/ (build.txt, lint.txt, tests.txt)            │
│    - Pretty-print summary to terminal                           │
└─────────────────────────────────────────────────────────────────┘
```

### State Machine (per eval run)

```
                    ┌─────────┐
                    │  INIT   │
                    └────┬────┘
                         │
                         ▼
                    ┌─────────┐
               ┌────│ SANDBOX │────┐
               │    │ CREATE  │    │
               │    └────┬────┘    │
               │         │         │
            fail         │      success
               │         ▼         │
               │    ┌─────────┐    │
               │    │  SETUP  │────┤
               │    └────┬────┘    │
               │         │         │
               │      success      │
               │         │         │
               │         ▼         │
               │    ┌─────────┐    │
               │    │  AGENT  │────┤
               │    └────┬────┘    │
               │         │         │
               │      complete     │
               │         │         │
               │         ▼         │
               │    ┌─────────┐    │
               │    │ SCRIPTS │────┤
               │    └────┬────┘    │
               │         │         │
               │    pass │ fail    │
               │         │    └────┤
               │         ▼         │
               │    ┌─────────┐    │
               │    │  TESTS  │────┤
               │    └────┬────┘    │
               │         │         │
               │    pass │ fail    │
               │         │    └────┤
               │         ▼         ▼
               │    ┌─────────┐┌─────────┐
               │    │ PASSED  ││ FAILED  │
               │    └────┬────┘└────┬────┘
               │         │         │
               └─────────┼─────────┘
                         │
                         ▼
                    ┌─────────┐
                    │ CLEANUP │
                    └─────────┘
```

---

## Experiment Configuration

### Full Configuration Schema

```ts
interface ExperimentConfig {
  /**
   * Which agent CLI to use for running evals.
   *
   * Currently supported:
   * - 'claude-code': Anthropic's Claude Code CLI
   *
   * @default 'claude-code'
   */
  agent: 'claude-code'

  /**
   * Model to pass to the agent.
   *
   * Available models depend on the agent:
   * - claude-code: 'opus', 'sonnet', 'haiku'
   *
   * @default 'opus'
   */
  model: string

  /**
   * Which evals to run from the evals/ directory.
   *
   * Accepts:
   * - undefined: run all evals
   * - string: run single eval by exact name
   * - string[]: run multiple evals by exact name
   * - (name: string) => boolean: run evals matching predicate
   *
   * @default undefined (all evals)
   *
   * @example
   * // Run all evals
   * evals: undefined
   *
   * @example
   * // Run single eval
   * evals: 'add-button'
   *
   * @example
   * // Run multiple evals
   * evals: ['add-button', 'fix-auth']
   *
   * @example
   * // Run evals matching pattern
   * evals: (name) => name.startsWith('auth-')
   */
  evals?: string | string[] | ((name: string) => boolean)

  /**
   * Number of times to run each eval.
   *
   * Multiple runs allow measuring reliability/pass rate.
   * Results are aggregated in summary.json.
   *
   * @default 1
   */
  runs: number

  /**
   * Whether to stop running after the first successful run.
   *
   * - true: Stop early on first pass (saves time/cost)
   * - false: Run all attempts (measure true pass rate)
   *
   * When earlyExit is true and an eval passes, remaining
   * runs are skipped and summary shows attemptsUntilPass.
   *
   * @default true
   */
  earlyExit: boolean

  /**
   * npm scripts that must exit 0 before tests run.
   *
   * Scripts are run in order after the agent completes.
   * Each script runs as: `npm run <script>`
   *
   * Execution stops on first failure (non-zero exit).
   *
   * Common scripts:
   * - 'build': Compile TypeScript, bundle assets
   * - 'lint': Run ESLint, Prettier
   * - 'typecheck': Run tsc --noEmit
   *
   * @default []
   *
   * @example
   * scripts: ['build', 'lint', 'typecheck']
   */
  scripts: string[]

  /**
   * Setup function that runs before the agent starts.
   *
   * Use this to:
   * - Install skills: await sandbox.exec('npx skills add owner/repo')
   * - Configure environment
   * - Pre-populate files
   *
   * If setup throws or returns rejected promise, the eval
   * fails immediately without running the agent.
   *
   * The sandbox parameter provides the same interface used
   * in EVAL.ts tests.
   *
   * @default undefined (no setup)
   *
   * @example
   * setup: async (sandbox) => {
   *   await sandbox.exec('npx skills add vercel/next-skill')
   * }
   */
  setup?: (sandbox: Sandbox) => Promise<void>
}
```

### Configuration Examples

**Minimal Configuration:**

```ts
// experiments/baseline.ts
export default {}

// Equivalent to:
// {
//   agent: 'claude-code',
//   model: 'opus',
//   evals: undefined,      // all evals
//   runs: 1,
//   earlyExit: true,
//   scripts: [],
//   setup: undefined,
// }
```

**Skill Testing Configuration:**

```ts
// experiments/with-next-skill.ts
export default {
  agent: 'claude-code',
  model: 'opus',
  evals: (name) => name.startsWith('nextjs-'),
  runs: 10,
  earlyExit: false,  // measure true pass rate
  scripts: ['build', 'lint'],
  setup: async (sandbox) => {
    await sandbox.exec('npx skills add vercel/next-skill')
  },
}
```

**Model Comparison Configuration:**

```ts
// experiments/sonnet-baseline.ts
export default {
  agent: 'claude-code',
  model: 'sonnet',
  runs: 5,
  earlyExit: false,
}

// experiments/opus-baseline.ts
export default {
  agent: 'claude-code',
  model: 'opus',
  runs: 5,
  earlyExit: false,
}
```

---

## Eval Fixtures

### Directory Structure

Each eval is a complete Node.js project with two special files:

```
evals/
└── add-button/                # Eval name (directory name)
    ├── src/                   # Source code (starting state)
    │   ├── App.tsx
    │   ├── components/
    │   │   └── Header.tsx
    │   └── index.ts
    ├── package.json           # Dependencies and scripts
    ├── tsconfig.json          # TypeScript config (optional)
    ├── PROMPT.md              # Task for the agent
    └── EVAL.ts                # Validation tests
```

### PROMPT.md Specification

**Purpose:** Define the task for the AI agent to complete.

**Format:** Markdown (rendered as plain text to agent)

**Best Practices:**
- Write clear, specific requirements
- Include acceptance criteria
- Mention constraints (e.g., "don't modify existing tests")
- Reference specific files when helpful

**Example:**

```md
# Add Logout Button

Add a logout button to the application header.

## Requirements

1. Add a "Logout" button to the top-right corner of the Header component
2. The button should:
   - Have the text "Logout"
   - Use the existing Button component from `src/components/Button.tsx`
   - Call the `logout()` function from `src/auth.ts` when clicked
3. After logout, redirect the user to `/login`

## Constraints

- Do not modify the existing test files
- Maintain the current styling approach (Tailwind CSS)

## Files to Reference

- `src/components/Header.tsx` - Where to add the button
- `src/components/Button.tsx` - Existing button component to use
- `src/auth.ts` - Contains the logout function
```

### EVAL.ts Specification

**Purpose:** Validate that the agent completed the task correctly.

**Format:** TypeScript vitest tests

**Runtime Environment:**
- Runs inside the sandbox after agent completes
- Has access to global `sandbox` object
- Full vitest API available (test, expect, describe, etc.)

**Example:**

```ts
// evals/add-button/EVAL.ts
import { sandbox } from '@vercel/eval-framework'
import { test, expect, describe } from 'vitest'

describe('logout button implementation', () => {
  test('Header.tsx contains logout button', async () => {
    const content = await sandbox.readFile('src/components/Header.tsx')
    expect(content).toMatch(/logout/i)
    expect(content).toMatch(/<Button/)
  })

  test('logout button calls logout function', async () => {
    const content = await sandbox.readFile('src/components/Header.tsx')
    expect(content).toMatch(/import.*logout.*from.*auth/i)
    expect(content).toMatch(/onClick.*logout/)
  })

  test('application builds without errors', async () => {
    const result = await sandbox.exec('npm run build')
    expect(result.exitCode).toBe(0)
  })

  test('application starts and responds', async () => {
    // Start dev server in background
    sandbox.exec('npm run dev -- --port 3456 &')

    // Wait for server to start
    await new Promise(resolve => setTimeout(resolve, 5000))

    // Check server responds
    const health = await sandbox.exec('curl -s http://localhost:3456')
    expect(health.exitCode).toBe(0)
  })

  test('no console errors on page load', async () => {
    // This would require a headless browser setup
    // Example pattern for more complex validation
    const result = await sandbox.exec(`
      npx playwright test --reporter=json tests/no-console-errors.spec.ts
    `)
    expect(result.exitCode).toBe(0)
  })
})
```

**Sandbox API in EVAL.ts:**

```ts
// Execute shell command
const result = await sandbox.exec('npm run build')
// result.stdout: string
// result.stderr: string
// result.exitCode: number

// Read file content
const content = await sandbox.readFile('src/App.tsx')

// Write file
await sandbox.writeFile('config.json', JSON.stringify({ debug: true }))

// Find files by glob pattern
const tsFiles = await sandbox.glob('**/*.tsx')
const allFiles = await sandbox.glob()  // default: '**/*'
```

---

## Sandbox Environment

### Vercel Sandbox Architecture

The framework exclusively uses Vercel Sandbox (Firecracker MicroVMs) for execution.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Vercel Sandbox Infrastructure               │
├─────────────────────────────────────────────────────────────────┤
│  ┌────────────────────────────────────────────────────────┐     │
│  │                   Firecracker MicroVM                   │     │
│  │  ┌─────────────────────────────────────────────────┐   │     │
│  │  │              Amazon Linux 2023                   │   │     │
│  │  │                                                  │   │     │
│  │  │  Pre-installed:                                  │   │     │
│  │  │  • Node.js 24                                    │   │     │
│  │  │  • npm, pnpm                                     │   │     │
│  │  │  • git                                           │   │     │
│  │  │  • Common build tools (gcc, make, python3)       │   │     │
│  │  │  • curl, wget                                    │   │     │
│  │  │                                                  │   │     │
│  │  │  Writable directory: /vercel/sandbox/            │   │     │
│  │  │                                                  │   │     │
│  │  │  Network: Outbound only (npm registry, etc.)     │   │     │
│  │  │                                                  │   │     │
│  │  │  Resources:                                      │   │     │
│  │  │  • CPU: 2 vCPUs                                  │   │     │
│  │  │  • Memory: 4GB                                   │   │     │
│  │  │  • Disk: 10GB                                    │   │     │
│  │  │                                                  │   │     │
│  │  └─────────────────────────────────────────────────┘   │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
│  Isolation: Each eval run gets a fresh VM                        │
│  Cleanup: VMs destroyed after eval completes                     │
│  Concurrency: Multiple VMs run in parallel                       │
└─────────────────────────────────────────────────────────────────┘
```

### Why No Local Sandbox?

The framework intentionally does not support local execution:

1. **Security Risk**
   - AI agents execute arbitrary shell commands
   - Without isolation, agents could read/write/delete any file on the host
   - Could access sensitive data, credentials, SSH keys
   - Could install malware or cryptominers

2. **Permission Complexity**
   - Local execution would require dangerous permission bypasses
   - Different OS/filesystem permissions add complexity
   - Would need to carefully sandbox file access, network, processes
   - Easy to get wrong, hard to verify correctness

3. **Reproducibility**
   - Local environments differ (installed packages, OS version, paths)
   - "Works on my machine" becomes "passes on my machine"
   - Vercel Sandbox provides consistent Amazon Linux 2023 environment
   - Same Node.js version, same tools, same behavior

4. **Resource Isolation**
   - Local execution competes with user's other processes
   - No memory/CPU limits without container setup
   - Sandbox provides dedicated resources per eval

### Sandbox Interface

```ts
interface Sandbox {
  /**
   * Execute a shell command in the sandbox.
   *
   * @param cmd - Command to execute
   * @returns Promise with stdout, stderr, and exit code
   *
   * @example
   * const result = await sandbox.exec('npm run build')
   * if (result.exitCode !== 0) {
   *   console.error(result.stderr)
   * }
   */
  exec(cmd: string): Promise<{
    stdout: string
    stderr: string
    exitCode: number
  }>

  /**
   * Read a file from the sandbox.
   *
   * @param path - Relative path from sandbox root
   * @returns Promise with file contents as string
   * @throws Error if file doesn't exist
   *
   * @example
   * const content = await sandbox.readFile('src/App.tsx')
   */
  readFile(path: string): Promise<string>

  /**
   * Write a file to the sandbox.
   *
   * @param path - Relative path from sandbox root
   * @param content - File content to write
   *
   * @example
   * await sandbox.writeFile('config.json', '{"debug": true}')
   */
  writeFile(path: string, content: string): Promise<void>

  /**
   * Find files matching a glob pattern.
   *
   * @param pattern - Glob pattern (default: '**\/*')
   * @returns Promise with array of matching file paths
   *
   * @example
   * const tsFiles = await sandbox.glob('**\/*.tsx')
   */
  glob(pattern?: string): Promise<string[]>

  /**
   * Destroy the sandbox and free resources.
   * Called automatically after eval completes.
   */
  destroy(): Promise<void>
}
```

---

## Agent Execution

### Claude Code Integration

The framework spawns the `claude-code` CLI inside the sandbox:

```bash
claude-code \
  --model opus \
  --print \
  --output-format stream-json \
  --dangerously-skip-permissions \
  <<< "$PROMPT_CONTENT"
```

**Flags:**
- `--model opus`: Use specified model
- `--print`: Print conversation to stdout
- `--output-format stream-json`: Output as JSONL for transcript
- `--dangerously-skip-permissions`: Auto-approve all tool calls (sandboxed environment)

### Agent Transcript Format

Claude Code outputs JSONL (JSON Lines) format:

```jsonl
{"type":"system","timestamp":"2026-01-26T12:00:00Z","content":"Starting Claude Code..."}
{"type":"user","timestamp":"2026-01-26T12:00:01Z","content":"Add a logout button..."}
{"type":"assistant","timestamp":"2026-01-26T12:00:05Z","content":"I'll add a logout button..."}
{"type":"tool_use","timestamp":"2026-01-26T12:00:06Z","tool":"read_file","input":{"path":"src/Header.tsx"}}
{"type":"tool_result","timestamp":"2026-01-26T12:00:06Z","content":"export function Header()..."}
{"type":"assistant","timestamp":"2026-01-26T12:00:10Z","content":"I'll modify the Header..."}
{"type":"tool_use","timestamp":"2026-01-26T12:00:11Z","tool":"write_file","input":{"path":"src/Header.tsx","content":"..."}}
{"type":"tool_result","timestamp":"2026-01-26T12:00:11Z","content":"File written successfully"}
{"type":"assistant","timestamp":"2026-01-26T12:00:15Z","content":"Done! I've added the logout button."}
{"type":"end","timestamp":"2026-01-26T12:00:15Z","status":"success"}
```

### Agent Timeout

- Default timeout: 10 minutes
- On timeout: SIGTERM → wait 5s → SIGKILL
- Timeout triggers eval failure

---

## Test Execution

### Vitest Integration

The framework uses vitest for running EVAL.ts tests:

**Injected Configuration:**

```ts
// vitest.config.ts (written to sandbox)
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals: true,
    include: ['EVAL.ts'],
    testTimeout: 60000,
    hookTimeout: 30000,
    reporters: ['json', 'default'],
    outputFile: {
      json: './vitest-results.json',
    },
  },
})
```

**Test Execution:**

```bash
npx vitest run EVAL.ts --reporter=json --outputFile=vitest-results.json
```

### Test Result Parsing

The framework parses vitest JSON output:

```json
{
  "numTotalTests": 5,
  "numPassedTests": 3,
  "numFailedTests": 2,
  "testResults": [
    {
      "name": "logout button exists",
      "status": "passed",
      "duration": 150
    },
    {
      "name": "app builds successfully",
      "status": "failed",
      "duration": 5200,
      "failureMessages": ["Expected exit code 0 but got 1"]
    }
  ]
}
```

---

## Scoring System

### Binary Pass/Fail

Every eval run results in exactly one of two outcomes:

- **Pass (1)**: All conditions met
- **Fail (0)**: Any condition failed

### Pass Conditions

An eval passes if and only if ALL of the following succeed:

```
PASS = setup.passed AND scripts.allPassed AND tests.allPassed
```

| Phase | Pass Condition |
|-------|----------------|
| Setup | Function completes without throwing |
| Scripts | All scripts exit with code 0 |
| Tests | All vitest tests pass |

### Failure Cascade

```
Setup fails    →  Skip agent, scripts, tests  →  FAIL
Agent timeout  →  Continue to scripts, tests  →  (may still fail)
Script fails   →  Skip remaining scripts      →  FAIL
Test fails     →  Continue remaining tests    →  FAIL
```

### Pass Rate Calculation

For experiments with multiple runs:

```
passRate = passedRuns / totalRuns
```

Example: 7 passes out of 10 runs = 0.7 (70%)

---

## Results and Reporting

### Directory Structure

```
results/
└── {experiment-name}/
    └── {timestamp}/
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

**Naming Conventions:**
- Experiment name: Filename without extension (`with-skill.ts` → `with-skill`)
- Timestamp: ISO8601 UTC, colons replaced with dashes (`2026-01-26T12-00-00Z`)

### result.json Schema

```json
{
  "eval": "add-button",
  "run": 1,
  "passed": false,
  "duration": 45200,
  "timestamp": "2026-01-26T12:00:00Z",
  "config": {
    "agent": "claude-code",
    "model": "opus"
  },
  "setup": {
    "passed": true,
    "duration": 1200
  },
  "agent": {
    "completed": true,
    "duration": 28000,
    "exitCode": 0
  },
  "scripts": {
    "build": {
      "passed": true,
      "duration": 12300,
      "exitCode": 0,
      "output": "./outputs/build.txt"
    },
    "lint": {
      "passed": false,
      "duration": 2100,
      "exitCode": 1,
      "output": "./outputs/lint.txt"
    }
  },
  "tests": {
    "passed": false,
    "total": 5,
    "passedCount": 3,
    "failedCount": 2,
    "failures": [
      "logout button exists",
      "redirects to /login after logout"
    ],
    "duration": 1600,
    "output": "./outputs/tests.txt"
  },
  "transcript": "./transcript.jsonl"
}
```

### summary.json Schema

```json
{
  "eval": "add-button",
  "config": {
    "agent": "claude-code",
    "model": "opus",
    "runs": 10,
    "earlyExit": true
  },
  "results": {
    "total": 10,
    "passed": 7,
    "failed": 3,
    "passRate": 0.7
  },
  "timing": {
    "meanDuration": 45200,
    "minDuration": 32100,
    "maxDuration": 58400,
    "stddev": 8300
  },
  "earlyExit": {
    "enabled": true,
    "stoppedEarly": true,
    "attemptsUntilPass": 3
  },
  "failures": {
    "setup": 0,
    "scripts": 1,
    "tests": 2
  }
}
```

### Terminal Output

After each eval completes, the framework pretty-prints a summary:

```
┌─────────────────────────────────────────────────────────────────┐
│  add-button                                                     │
│                                                                 │
│  Result:    ✓ 7/10 passed (70.0%)                               │
│  Duration:  Mean 45.2s (σ 8.3s)                                 │
│  Early Exit: Stopped after 3 attempts (first pass)              │
│                                                                 │
│  Failures by phase:                                             │
│    Setup:   0                                                   │
│    Scripts: 1 (lint)                                            │
│    Tests:   2                                                   │
│                                                                 │
│  Details: results/with-skill/2026-01-26T12-00-00Z/add-button/   │
└─────────────────────────────────────────────────────────────────┘
```

---

## CLI Interface

### Commands

```bash
# Run an experiment
npx eval <experiment-path>
npx eval experiments/my-experiment.ts

# Initialize a new eval project
npx eval init

# List available evals
npx eval list

# Show help
npx eval --help
```

### Init Command

Creates a new eval project with example files:

```bash
npx eval init
```

**Output:**

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
├── .gitignore
└── package.json
```

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | All evals passed |
| 1 | One or more evals failed |
| 2 | Configuration error |
| 3 | Missing credentials |

---

## Error Handling

### Error Categories

| Category | Handling | Recovery |
|----------|----------|----------|
| Config validation | Fail fast with descriptive message | User fixes config |
| Missing credentials | Fail fast, prompt for setup | User adds to .env |
| Eval fixture invalid | Warn and skip eval | User fixes fixture |
| Sandbox creation fails | Retry once, then fail eval | System recovers |
| Setup throws | Fail eval, continue others | User fixes setup |
| Agent timeout | Fail eval, continue others | Increase timeout |
| Network error | Retry with backoff | Transient |

### Error Messages

```ts
// Config validation
"Config error: 'model' must be 'opus', 'sonnet', or 'haiku', got 'gpt-4'"

// Missing credentials
"Missing VERCEL_TOKEN or VERCEL_OIDC_TOKEN. Add to .env file."

// Invalid fixture
"Warning: evals/add-button missing EVAL.ts, skipping"

// Sandbox error
"Sandbox creation failed for add-button (run 3): VM limit exceeded. Retrying..."

// Setup failure
"Setup failed for add-button: npx skills add failed with exit code 1"
```

---

## Concurrency Model

### Parallel Execution

All evals and runs execute concurrently (limited by sandbox capacity):

```
Experiment: 2 evals × 5 runs = 10 total executions

Time ─────────────────────────────────────────────────────►

VM1:  [add-button run-1]──────────────────────────────────
VM2:  [add-button run-2]──────────────────────────────────
VM3:  [add-button run-3]─────────────────────────────
VM4:  [add-button run-4]──────────────────────────────────
VM5:  [add-button run-5]────────────────────────────
VM6:  [fix-auth run-1]────────────────────────────────────
VM7:  [fix-auth run-2]──────────────────────────────
VM8:  [fix-auth run-3]────────────────────────────────────
VM9:  [fix-auth run-4]───────────────────────────────
VM10: [fix-auth run-5]────────────────────────────────────
```

### Early Exit Handling

When `earlyExit: true` and an eval passes:

```
add-button run-1: FAIL (continue)
add-button run-2: FAIL (continue)
add-button run-3: PASS (cancel run-4, run-5)
add-button run-4: CANCELLED
add-button run-5: CANCELLED

Summary: 1/3 passed (stopped after 3 attempts)
```

### Concurrency Limits

- **Per-account sandbox limit:** Configurable (default varies)
- **Framework default:** min(evals × runs, sandbox_limit)
- **Backpressure:** Queue excess work, process as VMs free up

---

## Security Considerations

### Threat Model

| Threat | Mitigation |
|--------|------------|
| Agent escapes sandbox | Firecracker VM isolation |
| Agent accesses credentials | Credentials in host .env, not in sandbox |
| Agent makes external requests | Network egress allowed (required for npm) |
| Malicious eval fixture | User-authored, runs in their account |
| Supply chain attack | npm audit, lockfiles |

### Credential Handling

```
Host Machine                    Sandbox VM
┌─────────────────┐            ┌─────────────────┐
│ .env            │            │ /vercel/sandbox │
│ ANTHROPIC_KEY=x │──inject───►│ ANTHROPIC_KEY=x │
│ VERCEL_TOKEN=y  │            │ (not present)   │
└─────────────────┘            └─────────────────┘
```

- `ANTHROPIC_API_KEY`: Injected into sandbox for agent
- `VERCEL_TOKEN`: Used by framework, NOT in sandbox

---

## Environment Variables

### Required Variables

```bash
# .env

# Anthropic API key for Claude Code agent
# Required for running evals
ANTHROPIC_API_KEY=sk-ant-api03-...

# Vercel authentication (one of these required)
# Used to create sandbox VMs
VERCEL_TOKEN=...
# OR
VERCEL_OIDC_TOKEN=...
```

### Optional Variables

```bash
# Override default model
EVAL_DEFAULT_MODEL=sonnet

# Override default timeout (ms)
EVAL_AGENT_TIMEOUT=600000

# Enable debug logging
EVAL_DEBUG=true
```

---

## Future Considerations

### Planned Features

1. **Additional Agents**
   - Cursor agent support
   - Copilot agent support
   - Custom agent adapters

2. **Additional Test Frameworks**
   - Jest support
   - Playwright for E2E tests

3. **Enhanced Reporting**
   - HTML report generation
   - Comparison between experiments
   - Trend tracking over time

4. **Eval Library**
   - Shared eval fixtures (npm packages)
   - Community contributions
   - Versioned evals for benchmarking

### Extension Points

```ts
// Future: Custom agent adapter
interface AgentAdapter {
  name: string
  spawn(sandbox: Sandbox, prompt: string, config: AgentConfig): Promise<AgentResult>
}

// Future: Custom reporter
interface Reporter {
  onEvalStart(eval: Eval): void
  onRunComplete(result: RunResult): void
  onEvalComplete(summary: Summary): void
  onExperimentComplete(results: ExperimentResults): void
}
```
