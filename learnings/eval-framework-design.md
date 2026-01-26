# How to Use @vercel/eval-framework

**Date:** 2026-01-26

Test whether an AI coding agent can complete real tasks in your codebase.

---

## Quick Start (5 steps)

```bash
# 1. Create a new eval project
npx eval init

# 2. Set your API key
export ANTHROPIC_API_KEY=your-key

# 3. Run the example eval
npx eval configs/default.ts
```

That's it. You now have results in `results/default/`.

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
├── configs/
│   └── default.ts         # Which evals to run, how many times
└── results/               # Results appear here
```

---

## Step 1: Write Your Task

Create `PROMPT.md` with instructions for the agent:

```md
Add a logout button to the header.

Requirements:
- Button should be in the top right
- Clicking it should clear the session and redirect to /login
```

Write it like you'd write for a junior developer. Be specific about what you want.

---

## Step 2: Write Your Tests

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

The `sandbox` object lets you:
- `sandbox.exec('command')` — run shell commands
- `sandbox.readFile('path')` — read files
- `sandbox.writeFile('path', 'content')` — write files
- `sandbox.glob('**/*.tsx')` — find files

---

## Step 3: Create a Config

Create `configs/my-test.ts`:

```ts
export default {
  evals: ['add-button'],  // which evals to run
  runs: 5,                // run each eval 5 times
  model: 'sonnet',        // which Claude model to use
}
```

### Config Options

| Option | What it does | Default |
|--------|--------------|---------|
| `evals` | Which evals to run | all evals |
| `runs` | How many times to run each eval | 1 |
| `model` | Claude model (`'sonnet'`, `'opus'`) | `'sonnet'` |
| `scripts` | npm scripts that must pass (e.g., `['build', 'lint']`) | none |
| `setup` | Code to run before the agent starts | none |
| `earlyExit` | Stop after first success? | `true` |

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

## Step 4: Run It

```bash
npx eval configs/my-test.ts
```

---

## Step 5: Check Results

Results appear in `results/<config-name>/<timestamp>/`:

```
results/my-test/2026-01-26T12-00-00Z/
├── add-button/
│   ├── run-1/
│   │   ├── result.json      # Did it pass? How long?
│   │   ├── transcript.jsonl # Full agent conversation
│   │   └── outputs/         # Build logs, test output
│   ├── run-2/
│   └── summary.json         # "7 out of 10 runs passed"
```

### Understanding summary.json

```json
{
  "eval": "add-button",
  "runs": 10,
  "passed": 7,
  "passRate": 0.7,
  "meanDuration": 45200
}
```

**Pass rate of 0.7 = the agent succeeds 70% of the time.**

---

## Common Patterns

### Run setup before the agent starts

```ts
export default {
  setup: async (sandbox) => {
    await sandbox.exec('npm install some-package')
  },
}
```

### Require build/lint to pass

```ts
export default {
  scripts: ['build', 'lint'],  // must exit 0
}
```

### Pass environment variables to the agent

```ts
export default {
  env: {
    DATABASE_URL: 'postgres://...',
  },
}
```

### Run all attempts (don't stop on first success)

```ts
export default {
  runs: 10,
  earlyExit: false,  // run all 10 even if some pass
}
```

---

## Where Does It Run?

The framework picks automatically:

| If you have... | It uses... |
|----------------|------------|
| `VERCEL_TOKEN` set | Vercel Sandbox (isolated VMs, runs in parallel) |
| No token | Local sandbox (temp directory on your machine) |

For production evals, use Vercel Sandbox. It's faster and isolated.

---

## Environment Variables

```bash
# Required
ANTHROPIC_API_KEY=your-anthropic-key

# Optional (enables Vercel Sandbox)
VERCEL_TOKEN=your-vercel-token
```

---

## How Scoring Works

An eval is **pass or fail**. No partial credit.

To pass, ALL of these must succeed:
1. Setup (if you defined one)
2. All scripts in `scripts` array
3. All vitest tests in EVAL.ts

---

## Troubleshooting

**"Why did my eval fail?"**
→ Check `transcript.jsonl` to see what the agent did

**"The agent succeeded but tests failed"**
→ Check `outputs/tests.txt` for test errors

**"Setup keeps failing"**
→ Check that your fixture has a valid `package.json`
