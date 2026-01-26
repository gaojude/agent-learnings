# How to Use @vercel/eval-framework

**Date:** 2026-01-26

Test whether an AI coding agent can complete real tasks in your codebase.

---

## Quick Start

```bash
# 1. Create a new eval project
npx eval init

# 2. Run the example eval
npx eval experiments/default.ts
```

Results appear in `results/default/` with a summary printed to your terminal:

```
┌─────────────────────────────────────────────────┐
│  add-button                                     │
│  ✓ 7/10 passed (70%)                            │
│  Mean duration: 45.2s                           │
│                                                 │
│  Details: results/default/2026-01-26T12-00-00Z/ │
└─────────────────────────────────────────────────┘
```

---

## What You're Building

An "eval" is just a test for an AI agent. You give it:
- A **task** (written in markdown)
- A **way to check** if it succeeded (written in vitest)

```
my-evals/
├── evals/
│   └── add-button/        # Your eval (a normal Node.js project)
│       ├── src/App.tsx    # Starting code the agent will modify
│       ├── package.json
│       ├── PROMPT.md      # "Add a logout button to the header"
│       └── EVAL.ts        # Tests to verify the agent did it right
├── experiments/
│   └── default.ts         # Which evals to run, how many times
├── .env                   # API keys
└── results/               # Results appear here
```

---

## Step 1: Set Up Your .env

Create a `.env` file in your project root:

```bash
# Required - for Anthropic API
ANTHROPIC_API_KEY=sk-ant-...

# Required - for Vercel Sandbox (isolated VMs)
VERCEL_TOKEN=your-vercel-token
# OR use OIDC token
VERCEL_OIDC_TOKEN=your-oidc-token
```

The framework reads these automatically. Never pass API keys manually.

---

## Step 2: Write Your Task

Create `PROMPT.md` with instructions for the agent:

```md
Add a logout button to the header.

Requirements:
- Button should be in the top right
- Clicking it should clear the session and redirect to /login
```

Write it like you'd write for a junior developer. Be specific about what you want.

---

## Step 3: Write Your Tests

Create `EVAL.ts` to check if the agent succeeded:

```ts
import { sandbox } from '@vercel/eval-framework'
import { test, expect } from 'vitest'

test('logout button exists in codebase', async () => {
  const files = await sandbox.glob('**/*.tsx')
  for (const file of files) {
    const content = await sandbox.readFile(file)
    if (/logout/i.test(content)) {
      expect(true).toBe(true)
      return
    }
  }
  expect.fail('No logout button found')
})

test('app still builds', async () => {
  const result = await sandbox.exec('npm run build')
  expect(result.exitCode).toBe(0)
})
```

**How the import works:** The framework injects `EVAL.ts` into the sandbox after the agent runs. The `@vercel/eval-framework` package provides a global `sandbox` instance that's pre-configured to interact with the current sandbox environment. You don't need to install it in your eval fixture—it's provided by the framework at runtime.

The `sandbox` object lets you:
- `sandbox.exec('command')` — run shell commands
- `sandbox.readFile('path')` — read files
- `sandbox.writeFile('path', 'content')` — write files
- `sandbox.glob('**/*.tsx')` — find files

---

## Step 4: Create an Experiment

Create `experiments/my-experiment.ts`:

```ts
export default {
  agent: 'claude-code',     // which agent to use (required concept)
  model: 'opus',            // model is secondary to agent choice
  evals: ['add-button'],    // which evals to run
  runs: 5,                  // run each eval 5 times
}
```

---

## Step 5: Run It

```bash
npx eval experiments/my-experiment.ts
```

---

## Step 6: Check Results

After each eval completes, the framework prints a summary:

```
┌─────────────────────────────────────────────────┐
│  add-button                                     │
│  ✓ 7/10 passed (70%)                            │
│  Mean duration: 45.2s                           │
│                                                 │
│  Details: results/my-experiment/2026-01-26.../  │
└─────────────────────────────────────────────────┘
```

Full results are in `results/<experiment-name>/<timestamp>/`:

```
results/my-experiment/2026-01-26T12-00-00Z/
├── add-button/
│   ├── run-1/
│   │   ├── result.json      # Did it pass? How long?
│   │   ├── transcript.jsonl # Full agent conversation
│   │   └── outputs/         # Build logs, test output
│   ├── run-2/
│   └── summary.json         # "7 out of 10 runs passed"
```

---

## Full Experiment Config Reference

```ts
// experiments/example.ts
export default {
  // AGENT (primary choice)
  // Which agent CLI to use for running evals
  // Currently supported: 'claude-code'
  agent: 'claude-code',

  // MODEL (secondary to agent)
  // Model to pass to the agent
  // Options depend on agent. For claude-code: 'opus', 'sonnet', 'haiku'
  // Default: 'opus'
  model: 'opus',

  // EVALS
  // Which evals to run from the evals/ folder
  // - undefined: run all evals
  // - string: run single eval by name
  // - string[]: run multiple evals by name
  // - (name) => boolean: run evals matching predicate
  // Default: all evals
  evals: ['add-button', 'fix-auth'],

  // RUNS
  // Number of times to run each eval
  // Results aggregated in summary.json
  // Default: 1
  runs: 10,

  // EARLY EXIT
  // Stop running after first successful run
  // Set to false to always run all attempts (useful for measuring pass rate)
  // Default: true
  earlyExit: false,

  // SCRIPTS
  // npm scripts that must exit 0 before tests run
  // These run `npm run <script>` in order after agent completes
  // Stops on first failure
  // Default: []
  scripts: ['build', 'lint', 'typecheck'],

  // SETUP
  // Function that runs before agent starts
  // Use this to install skills, configure the environment, etc.
  // If setup throws, eval fails immediately
  // Default: none
  setup: async (sandbox) => {
    // Example: Install a skill before the agent runs
    await sandbox.exec('npx skills add vercel/next-skill')
  },
}
```

### Config Options Summary Table

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `agent` | `'claude-code'` | `'claude-code'` | Which agent CLI to use |
| `model` | `string` | `'opus'` | Model for the agent (`'opus'`, `'sonnet'`, `'haiku'`) |
| `evals` | `string \| string[] \| (name) => boolean` | all | Which evals to run |
| `runs` | `number` | `1` | Number of runs per eval |
| `earlyExit` | `boolean` | `true` | Stop after first success |
| `scripts` | `string[]` | `[]` | npm scripts that must pass (runs `npm run <script>`) |
| `setup` | `(sandbox) => Promise<void>` | none | Setup function before agent starts |

### Selecting Evals

```ts
// Run all evals
evals: undefined

// Run one eval
evals: 'add-button'

// Run specific evals
evals: ['add-button', 'fix-auth']

// Run evals matching a pattern
evals: (name) => name.startsWith('auth-')
```

---

## Common Patterns

### Install a skill before the agent runs

```ts
export default {
  setup: async (sandbox) => {
    await sandbox.exec('npx skills add vercel/next-skill')
  },
}
```

### Require build/lint/typecheck to pass

```ts
export default {
  // These run as `npm run build`, `npm run lint`, etc.
  scripts: ['build', 'lint', 'typecheck'],
}
```

### Run all attempts (measure true pass rate)

```ts
export default {
  runs: 10,
  earlyExit: false,  // run all 10 even if some pass
}
```

---

## How Scoring Works

An eval is **pass or fail**. No partial credit.

To pass, ALL of these must succeed:
1. Setup function (if defined)
2. All npm scripts in `scripts` array
3. All vitest tests in EVAL.ts

---

## FAQ

### Why is there no local sandbox option?

Local sandbox would run agent code directly on your machine without isolation. This is dangerous because:

1. **Security risk** — AI agents can execute arbitrary commands. Without isolation, a misbehaving agent could modify or delete files outside the eval directory.
2. **Permission headaches** — Local execution requires careful permission management that varies by OS.
3. **Non-reproducible results** — Local environment differences (installed packages, OS version, etc.) make results inconsistent.

Vercel Sandbox provides isolated Firecracker MicroVMs that are ephemeral and auto-cleanup. This is the only supported execution environment.

### How does the EVAL.ts import work?

When you write `import { sandbox } from '@vercel/eval-framework'`, you don't need to install this package in your eval fixture. The framework:

1. Copies your eval folder to the sandbox (excluding `PROMPT.md` and `EVAL.ts`)
2. Runs the agent with your prompt
3. Injects `EVAL.ts` into the sandbox
4. Provides the `sandbox` global at runtime when vitest executes

The import is resolved by the framework's test harness, not by your fixture's node_modules.

---

## Troubleshooting

**"Why did my eval fail?"**
→ Check `transcript.jsonl` to see what the agent did

**"The agent succeeded but tests failed"**
→ Check `outputs/tests.txt` for test errors

**"Setup keeps failing"**
→ Check that your fixture has a valid `package.json`

**"Missing VERCEL_TOKEN"**
→ Add `VERCEL_TOKEN` or `VERCEL_OIDC_TOKEN` to your `.env` file
