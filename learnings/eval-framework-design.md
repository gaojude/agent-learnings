# How to Use @vercel/eval-framework

**Date:** 2026-01-26

Test whether an AI coding agent can complete real tasks in your codebase.

---

## What is This Framework? (Start Here if You're New)

When you use an AI coding agent (like Claude Code) to write code, how do you know if it actually works well? You could test it manually, but that's slow and inconsistent.

This framework lets you **automatically test AI agents** by:
1. Giving them a coding task
2. Letting them attempt it (in a safe, isolated environment)
3. Running tests to verify they did it correctly
4. Repeating this process multiple times to measure reliability

**Why "eval"?** The term "eval" (short for "evaluation") is commonly used in AI/ML to mean "testing how well something performs." In this context, an eval = one test case for an AI agent.

---

## Quick Start

```bash
# 1. Create a new eval project
npx eval init

# 2. Run the example eval
npx eval experiments/default.ts
```

Results appear in `results/default/` with a terminal summary showing pass rates and execution times.

---

## Understanding the Project Structure

Here's what an eval project looks like:

```
my-evals/
├── evals/                    # All your test cases live here
│   └── add-button/           # One test case (a mini Node.js project)
│       ├── src/App.tsx       # Starting code the agent will modify
│       ├── package.json      # Dependencies for this test case
│       ├── PROMPT.md         # Instructions you give to the agent
│       └── EVAL.ts           # Tests to verify the agent succeeded
├── experiments/
│   └── default.ts            # Configuration: what to test and how
├── .env                      # Your API keys (keep secret!)
└── results/                  # Test results appear here automatically
```

**Key terms:**
- **Eval** = One test case, containing a task and verification tests
- **Experiment** = A configuration that says "run these evals X times with this agent"
- **Sandbox** = An isolated virtual machine where the agent runs (so it can't affect your real system)

---

## Step-by-Step Guide

### Step 1: Set Up Your API Keys

Create a `.env` file in your project root with your credentials:

```bash
# Required - lets the framework talk to Claude
ANTHROPIC_API_KEY=sk-ant-...

# Required - for Vercel Sandbox (the isolated VMs where agents run)
VERCEL_TOKEN=your-vercel-token
# OR use OIDC token if your organization uses that
VERCEL_OIDC_TOKEN=your-oidc-token
```

**Why do I need these?**
- `ANTHROPIC_API_KEY` - The AI agent needs this to function
- `VERCEL_TOKEN` - The sandbox (isolated VM) is hosted by Vercel

The framework reads these automatically. Never hardcode API keys in your code.

---

### Step 2: Write Your Task (PROMPT.md)

Create a `PROMPT.md` file with instructions for the agent. This is literally what the agent will see.

```markdown
Add a logout button to the header.

Requirements:
- Button should be in the top right
- Clicking it should clear the session and redirect to /login
```

**Tips for writing good prompts:**
- Be specific about what you want
- Write it like you would for a junior developer
- Include acceptance criteria when possible
- If something matters (like file location), say it explicitly

---

### Step 3: Write Your Tests (EVAL.ts)

Create an `EVAL.ts` file that checks if the agent actually did the task correctly.

```typescript
import { sandbox } from '@vercel/eval-framework'
import { test, expect } from 'vitest'

// Test 1: Check if the logout button exists somewhere in the code
test('logout button exists in codebase', async () => {
  // Find all TypeScript React files
  const files = await sandbox.glob('**/*.tsx')

  // Search each file for "logout"
  for (const file of files) {
    const content = await sandbox.readFile(file)
    if (/logout/i.test(content)) {
      expect(true).toBe(true)
      return  // Found it! Test passes.
    }
  }

  // If we get here, we never found it
  expect.fail('No logout button found')
})

// Test 2: Make sure the app still builds (agent didn't break anything)
test('app still builds', async () => {
  const result = await sandbox.exec('npm run build')
  expect(result.exitCode).toBe(0)  // Exit code 0 = success
})
```

**What is `sandbox`?**

The `sandbox` object is your interface to the isolated VM where the agent ran. It provides these methods:

| Method | What it does | Example |
|--------|--------------|---------|
| `sandbox.exec('command')` | Run a shell command | `sandbox.exec('npm run build')` |
| `sandbox.readFile('path')` | Read a file's contents | `sandbox.readFile('src/App.tsx')` |
| `sandbox.writeFile('path', 'content')` | Write to a file | `sandbox.writeFile('test.txt', 'hello')` |
| `sandbox.glob('pattern')` | Find files matching a pattern | `sandbox.glob('**/*.tsx')` |

**Note:** You don't need to install `@vercel/eval-framework` in your eval folder. The framework provides it automatically when running tests.

---

### Step 4: Create an Experiment Configuration

Create `experiments/my-experiment.ts` to define how to run your evals:

```typescript
export default {
  agent: 'claude-code',    // Which AI agent to use
  model: 'opus',           // Which model (opus is most capable)
  evals: ['add-button'],   // Which evals to run (folder names from evals/)
  runs: 5,                 // Run each eval 5 times to measure reliability
}
```

**Why run multiple times?**

AI agents aren't deterministic—they might succeed sometimes and fail others. Running multiple times gives you a **pass rate** (e.g., "7 out of 10 attempts succeeded = 70% reliability").

---

### Step 5: Run Your Experiment

```bash
npx eval experiments/my-experiment.ts
```

The framework will:
1. Create an isolated sandbox for each run
2. Copy your eval's code into the sandbox
3. Give the agent your prompt
4. Let the agent work
5. Run your tests
6. Record the results

---

### Step 6: Check Your Results

After completion, you'll see a summary in the terminal:

```
┌─────────────────────────────────────────────────┐
│ add-button                                      │
│ ✓ 7/10 passed (70%)                             │
│ Mean duration: 45.2s                            │
│                                                 │
│ Details: results/my-experiment/2026-01-26.../   │
└─────────────────────────────────────────────────┘
```

Full results are saved to disk:

```
results/my-experiment/2026-01-26T12-00-00Z/
├── add-button/
│   ├── run-1/
│   │   ├── result.json        # Pass/fail status and timing
│   │   ├── transcript.jsonl   # Everything the agent said/did
│   │   └── outputs/           # Build logs, test output, etc.
│   ├── run-2/
│   │   └── ...
│   └── summary.json           # "7 out of 10 runs passed"
```

**Debugging tip:** If an eval fails, check `transcript.jsonl` to see exactly what the agent did. This is invaluable for understanding failures.

---

## Experiment Configuration Reference

Here are all the options you can use in your experiment config:

```typescript
export default {
  // REQUIRED: Which agent to use
  // Currently only 'claude-code' is supported
  agent: 'claude-code',

  // Which AI model the agent should use
  // Options: 'opus' (most capable), 'sonnet' (balanced), 'haiku' (fastest)
  // Default: 'opus'
  model: 'opus',

  // Which evals to run
  // Default: all evals in the evals/ folder
  evals: ['add-button', 'fix-auth'],

  // How many times to run each eval
  // Default: 1
  runs: 10,

  // Should we stop early if we get a success?
  // true = stop after first pass (useful for CI: "does it work at all?")
  // false = run all attempts (useful for measuring reliability)
  // Default: true
  earlyExit: false,

  // npm scripts that must pass AFTER the agent finishes
  // Runs `npm run <script>` for each, in order
  // Default: []
  scripts: ['build', 'lint', 'typecheck'],

  // Setup function that runs BEFORE the agent starts
  // Use this to configure the environment
  // Default: none
  setup: async (sandbox) => {
    await sandbox.exec('npx skills add vercel/next-skill')
  },
}
```

### Quick Reference Table

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `agent` | `'claude-code'` | `'claude-code'` | Which agent CLI to use |
| `model` | `string` | `'opus'` | Model for the agent |
| `evals` | `string \| string[] \| function` | all | Which evals to run |
| `runs` | `number` | `1` | Number of runs per eval |
| `earlyExit` | `boolean` | `true` | Stop after first success |
| `scripts` | `string[]` | `[]` | npm scripts that must pass |
| `setup` | `function` | none | Setup before agent starts |

### Different Ways to Select Evals

```typescript
// Run ALL evals in the evals/ folder
evals: undefined

// Run a single eval
evals: 'add-button'

// Run specific evals
evals: ['add-button', 'fix-auth']

// Run evals matching a pattern (using a filter function)
evals: (name) => name.startsWith('auth-')
```

---

## Common Patterns

### Pattern: Install a skill before the agent runs

```typescript
export default {
  setup: async (sandbox) => {
    await sandbox.exec('npx skills add vercel/next-skill')
  },
}
```

### Pattern: Require code quality checks to pass

```typescript
export default {
  // After the agent finishes, run these npm scripts
  // If any fail, the eval fails
  scripts: ['build', 'lint', 'typecheck'],
}
```

### Pattern: Measure true reliability (no early exit)

```typescript
export default {
  runs: 10,
  earlyExit: false,  // Run all 10 even if early ones pass
}
```

---

## How Scoring Works

Evals are **pass or fail**—there's no partial credit.

To pass, ALL of these must succeed:
1. Setup function (if you defined one)
2. All npm scripts in the `scripts` array (if you defined any)
3. All vitest tests in EVAL.ts

If any step fails, the entire eval run is marked as failed.

---

## Frequently Asked Questions

### Why can't I run this locally on my own machine?

The framework requires Vercel Sandbox (isolated virtual machines) for safety and consistency:

1. **Security** — AI agents execute code. Without isolation, a malfunctioning agent could delete files or cause damage on your real system.

2. **Reproducibility** — Your local machine has specific packages, OS settings, etc. Sandboxes are consistent, making results comparable across runs and between different people.

3. **Simplicity** — No need to manage permissions or worry about cleanup.

### How does the `import { sandbox }` work if I didn't install it?

You don't need to install `@vercel/eval-framework` in your eval fixture. Here's what happens:

1. Framework copies your eval folder to the sandbox (excluding `PROMPT.md` and `EVAL.ts`)
2. Agent runs and modifies the code
3. Framework injects `EVAL.ts` into the sandbox
4. Framework provides the `sandbox` object automatically when vitest runs

The import is resolved by the framework's test harness, not by your node_modules.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Why did my eval fail?" | Check `transcript.jsonl` to see what the agent did |
| "Agent seemed to succeed but tests failed" | Check `outputs/tests.txt` for test error details |
| "Setup keeps failing" | Verify your eval fixture has a valid `package.json` |
| "Missing VERCEL_TOKEN" | Add `VERCEL_TOKEN` or `VERCEL_OIDC_TOKEN` to your `.env` file |
