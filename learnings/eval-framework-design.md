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

## Prerequisites

Before you begin, make sure you have:

| Requirement | How to check | How to install |
|-------------|--------------|----------------|
| **Node.js 18+** | `node --version` | [nodejs.org](https://nodejs.org) |
| **npm 9+** | `npm --version` | Comes with Node.js |
| **Anthropic API key** | Check your [Anthropic Console](https://console.anthropic.com) | Sign up at anthropic.com |
| **Vercel account** | Check your [Vercel Dashboard](https://vercel.com/dashboard) | Sign up at vercel.com |

**Assumed knowledge:**
- Basic command line usage (running commands in a terminal)
- Basic understanding of JavaScript/TypeScript
- Familiarity with npm (installing packages, running scripts)
- Understanding of what environment variables are (`.env` files)

---

## Installation

```bash
# Create a new eval project (recommended for beginners)
npx eval init my-evals
cd my-evals

# Or add to an existing project
npm install --save-dev @vercel/eval-framework
```

---

## How It Works (Visual Overview)

Here's what happens when you run an eval:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         YOUR MACHINE                                    │
│                                                                         │
│  ┌─────────────────┐      ┌─────────────────┐      ┌────────────────┐  │
│  │ experiments/    │      │ evals/          │      │ .env           │  │
│  │ my-experiment.ts│      │ add-button/     │      │ API keys       │  │
│  └────────┬────────┘      └────────┬────────┘      └───────┬────────┘  │
│           │                        │                       │           │
│           └────────────────────────┼───────────────────────┘           │
│                                    │                                    │
│                          ┌─────────▼─────────┐                         │
│                          │  npx eval run     │                         │
│                          │  (framework CLI)  │                         │
│                          └─────────┬─────────┘                         │
└────────────────────────────────────┼────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     VERCEL SANDBOX (isolated VM)                        │
│                                                                         │
│   Step 1: Copy eval fixture ──────────────────────────────────────────▶ │
│                                                                         │
│   Step 2: Agent reads PROMPT.md ──────────────────────────────────────▶ │
│                                                                         │
│   Step 3: Agent modifies code ────────────────────────────────────────▶ │
│           (src/App.tsx, etc.)                                           │
│                                                                         │
│   Step 4: Run npm scripts ────────────────────────────────────────────▶ │
│           (build, lint, typecheck)                                      │
│                                                                         │
│   Step 5: Run EVAL.ts tests ──────────────────────────────────────────▶ │
│           (vitest)                                                      │
│                                                                         │
│   Step 6: Return results ─────────────────────────────────────────────▶ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         YOUR MACHINE                                    │
│                                                                         │
│   results/my-experiment/2026-01-26T12-00-00Z/                          │
│   ├── add-button/                                                       │
│   │   ├── run-1/result.json      ← Pass/fail + timing                  │
│   │   ├── run-1/transcript.jsonl ← Full agent conversation             │
│   │   └── summary.json           ← Aggregated pass rate                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Start

```bash
# 1. Create a new eval project
npx eval init my-evals
cd my-evals

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

### What Makes a Valid Eval Fixture

Each folder in `evals/` must contain these files:

| File | Required | Purpose |
|------|----------|---------|
| `PROMPT.md` | Yes | Instructions for the agent |
| `EVAL.ts` | Yes | Vitest tests to verify success |
| `package.json` | Yes | Must include `"type": "module"` and any dependencies |

Optional but common:
- Source files (`.ts`, `.tsx`, `.js`, etc.) that the agent will modify
- Config files (`tsconfig.json`, `.eslintrc.js`, `vite.config.ts`, etc.)
- Test fixtures or sample data

**Minimal `package.json` example:**

```json
{
  "name": "my-eval",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "test": "vitest run"
  },
  "dependencies": {},
  "devDependencies": {
    "typescript": "^5.0.0"
  }
}
```

### How Dependencies Are Installed

When the framework runs your eval:

1. Your fixture folder is copied to the sandbox (excluding `node_modules/`)
2. `npm install` runs automatically using your `package.json`
3. The agent receives PROMPT.md and starts working

**Important notes:**
- Do NOT include `node_modules/` in your fixture (it's ignored)
- Use `package-lock.json` if you need reproducible dependency versions
- The sandbox uses **Node.js v20.x LTS**
- All packages must be published to npm (no local or private packages)

**Tip:** Test your fixture locally first:
```bash
cd evals/my-eval
rm -rf node_modules
npm install && npm run build
```

### Complete Example: Your First Eval

Here's a complete, working eval you can copy to get started:

**evals/add-greeting/src/App.tsx**
```tsx
export function App() {
  return (
    <div>
      <h1>Hello World</h1>
    </div>
  )
}
```

**evals/add-greeting/PROMPT.md**
```markdown
Add a greeting message below the heading that says "Welcome, user!"
```

**evals/add-greeting/EVAL.ts**
```typescript
import { sandbox } from '@vercel/eval-framework'
import { test, expect } from 'vitest'

test('greeting message exists', async () => {
  const content = await sandbox.readFile('src/App.tsx')
  expect(content).toContain('Welcome, user!')
})

test('app still builds', async () => {
  const result = await sandbox.exec('npm run build')
  expect(result.exitCode, `Build failed: ${result.stderr}`).toBe(0)
})
```

**evals/add-greeting/package.json**
```json
{
  "name": "add-greeting",
  "type": "module",
  "scripts": { "build": "tsc" },
  "dependencies": { "react": "^18.0.0" },
  "devDependencies": { "typescript": "^5.0.0", "@types/react": "^18.0.0" }
}
```

**evals/add-greeting/tsconfig.json**
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "outDir": "dist"
  },
  "include": ["src"]
}
```

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

**How to get these:**
- `ANTHROPIC_API_KEY` - Create an account at [console.anthropic.com](https://console.anthropic.com), then go to Settings → API Keys → Create Key
- `VERCEL_TOKEN` - Log in at [vercel.com](https://vercel.com), go to Settings → Tokens → Create Token

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
- Keep prompts concise—typically 5-20 lines. Very long prompts consume tokens and may dilute key instructions
- If you need extensive context (e.g., API docs), put it in a separate file and reference it: "See API_DOCS.md for specifications"

**What the agent sees:**

When the eval starts, the agent receives:
1. The contents of `PROMPT.md` as its initial instruction (passed directly, not as a file)
2. Access to all files in your eval folder (except `PROMPT.md` and `EVAL.ts`)

The agent does NOT see `PROMPT.md` or `EVAL.ts` as files in the sandbox—they're excluded. The agent can explore the codebase using file reading and shell commands, just like a human developer would. It does NOT automatically see all file contents—it must choose to read them.

**What the agent can do:**
- Read and write files anywhere in the eval fixture
- Run shell commands (bash)
- Run npm scripts defined in package.json
- Install npm packages (via `npm install package-name`)

**What the agent cannot do:**
- Access the internet (external network requests fail)
- Access files outside the eval fixture directory
- Read environment variables from your local `.env` (sandbox is isolated)
- Persist data between runs (each run starts fresh)
- Connect to external databases or APIs

**Tip:** If certain files are critical context, mention them in your prompt: "See `src/App.tsx` for the current implementation."

---

### Step 3: Write Your Tests (EVAL.ts)

Create an `EVAL.ts` file that checks if the agent actually did the task correctly.

```typescript
import { sandbox } from '@vercel/eval-framework'
import { test, expect } from 'vitest'

// Test 1: Check if the logout button exists somewhere in the code
test('logout button exists in codebase', async () => {
  const files = await sandbox.glob('**/*.tsx')
  const filesWithLogout: string[] = []

  for (const file of files) {
    const content = await sandbox.readFile(file)
    if (/logout/i.test(content)) {
      filesWithLogout.push(file)
    }
  }

  // Provide a helpful error message if the test fails
  expect(
    filesWithLogout.length,
    `Expected 'logout' in at least one .tsx file. Searched ${files.length} files: ${files.join(', ')}`
  ).toBeGreaterThan(0)
})

// Test 2: Make sure the app still builds (agent didn't break anything)
test('app still builds', async () => {
  const result = await sandbox.exec('npm run build')

  // Include build output in error message for easier debugging
  expect(
    result.exitCode,
    `Build failed with exit code ${result.exitCode}.\nstderr: ${result.stderr}\nstdout: ${result.stdout}`
  ).toBe(0)
})
```

**What is `sandbox`?**

The `sandbox` object is your interface to the isolated VM where the agent ran. It provides these methods:

| Method | Returns | Description |
|--------|---------|-------------|
| `sandbox.exec('command')` | `{ exitCode: number, stdout: string, stderr: string }` | Run a shell command |
| `sandbox.readFile('path')` | `string` | Read file contents (throws if file doesn't exist) |
| `sandbox.writeFile('path', 'content')` | `void` | Write or overwrite a file |
| `sandbox.glob('pattern')` | `string[]` | Find files matching a glob pattern (`*` any chars, `**` recursive, `{a,b}` alternatives). Example: `**/*.{ts,tsx}` |

**Error handling:** File methods (`readFile`, `writeFile`) throw exceptions on failure. Wrap in try/catch if needed:

```typescript
try {
  const content = await sandbox.readFile('maybe-missing.txt')
} catch (err) {
  // File doesn't exist, handle gracefully
}
```

**`sandbox.exec()` behavior:** Unlike file methods, `exec()` does NOT throw on non-zero exit codes. It always returns the result object—check `exitCode` yourself:

```typescript
const result = await sandbox.exec('npm run build')
if (result.exitCode !== 0) {
  console.error('Build failed:', result.stderr)
}
```

**Important:** All sandbox methods are asynchronous. Always use `await`:

```typescript
// WRONG - test may pass incorrectly because you're not awaiting the result
const content = sandbox.readFile('src/App.tsx')  // Returns Promise, not string!

// CORRECT
const content = await sandbox.readFile('src/App.tsx')
```

**File paths:** All paths are relative to the eval fixture root—the folder containing your `PROMPT.md` and `package.json`. For example, if your structure is:

```
evals/add-button/
├── src/App.tsx
├── PROMPT.md
└── package.json
```

Then use `sandbox.readFile('src/App.tsx')`, not `sandbox.readFile('App.tsx')` or an absolute path.

**Note:** You don't need to install `@vercel/eval-framework` or `vitest` in your eval folder. The framework provides both automatically when running tests.

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

**When does the agent finish?** The agent decides when it's done—it signals completion after it believes the task is complete. There's no explicit "done" signal you need to add to your prompt; the agent determines completion autonomously. If the agent exceeds the `timeout`, the run fails.

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

### Understanding a Failed Run

When an eval fails, `result.json` shows what went wrong:

```json
{
  "status": "failed",
  "failedStep": "tests",
  "error": "Test 'logout button exists' failed: Expected 'logout' in at least one .tsx file",
  "duration": 52.3
}
```

The `transcript.jsonl` shows the agent's conversation (one JSON object per line):

```json
{"role": "assistant", "content": "I'll add a logout button to the header..."}
{"role": "tool", "name": "write_file", "input": {"path": "src/Header.tsx", ...}}
{"role": "assistant", "content": "Done! I've added the logout button."}
```

| Field | Description |
|-------|-------------|
| `role` | `"user"`, `"assistant"`, or `"tool"` |
| `content` | The message text (for user/assistant) |
| `name` | Tool name when `role` is `"tool"` (e.g., `"write_file"`, `"bash"`) |
| `input` | Tool input parameters when `role` is `"tool"` |

**Tip:** Use `jq` to filter transcripts: `cat transcript.jsonl | jq 'select(.role == "tool")'` shows only tool calls.

**Common failure patterns:**

| Symptom | Likely Cause | How to Fix |
|---------|--------------|------------|
| Agent edited the wrong file | Unclear prompt | Be more explicit about which file to modify |
| Agent misunderstood the task | Ambiguous requirements | Add specific acceptance criteria to PROMPT.md |
| Code doesn't compile | Agent introduced syntax errors | Check `outputs/build.txt`; add `scripts: ['build']` |
| Tests pass but feature is broken | Weak tests | Write tests that verify actual behavior |

---

## Experiment Configuration Reference

Here are all the options you can use in your experiment config:

```typescript
export default {
  // REQUIRED: Which agent to use
  // Currently only 'claude-code' is supported
  agent: 'claude-code',

  // Which AI model the agent should use (Claude model tiers)
  // - 'opus': Most capable, best for complex tasks, highest cost
  // - 'sonnet': Balanced capability and cost
  // - 'haiku': Fastest and cheapest, good for simple tasks
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
  // These run INSIDE the sandbox using your eval's package.json
  // IMPORTANT: Each script must exist in your eval's package.json
  // Runs `npm run <script>` for each, in order. Stops on first failure.
  // Default: []
  scripts: ['build', 'lint', 'typecheck'],

  // Maximum time (in seconds) for the agent to complete the task
  // If exceeded, the run fails with a timeout error
  // Default: 300 (5 minutes)
  timeout: 600,

  // Setup function that runs BEFORE the agent starts
  // Note: npm install runs automatically before setup() is called
  // The `sandbox` parameter has the same API as in EVAL.ts:
  // exec(), readFile(), writeFile(), glob()
  // Use this for additional configuration beyond package.json
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
| `timeout` | `number` | `300` | Max seconds for agent to complete |
| `scripts` | `string[]` | `[]` | npm scripts that must pass |
| `setup` | `function` | none | Setup before agent starts |

### Different Ways to Select Evals

```typescript
// Run ALL evals in the evals/ folder
// (scans top-level directories only; ignores hidden folders like .git)
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

**What are skills?** Skills are pre-built capabilities that extend what the agent can do. For example, `vercel/next-skill` teaches the agent Next.js-specific patterns and best practices. Skills are optional—the agent works without them, but adding relevant skills can improve success rates for domain-specific tasks.

### Pattern: Require code quality checks to pass

```typescript
export default {
  // After the agent finishes, run these npm scripts
  // If any fail, the eval fails
  scripts: ['build', 'lint', 'typecheck'],
}
```

**Note:** The `scripts` array runs npm scripts from your eval fixture's `package.json`—these are your fixture's *own* scripts. `EVAL.ts` is a *separate* test file that the framework injects. They serve different purposes:
- `scripts: ['test']` — Run the fixture's existing test suite (verifies the agent didn't break things)
- `EVAL.ts` — Verify the agent completed the assigned task correctly

You'll typically use both.
```

### Pattern: Measure true reliability (no early exit)

```typescript
export default {
  runs: 10,
  earlyExit: false,  // Run all 10 even if early ones pass
}
```

### Pattern: Test behavior, not implementation

AI agents may solve tasks in unexpected but valid ways. Write tests that verify the *outcome*, not the specific implementation:

```typescript
// BAD: Tests specific implementation details
test('logout button in header', async () => {
  const header = await sandbox.readFile('src/components/Header.tsx')
  expect(header).toContain('logout')  // Fails if agent puts it elsewhere!
})

// GOOD: Tests that the feature exists somewhere
test('logout functionality exists', async () => {
  const result = await sandbox.exec('grep -r "logout" src/')
  expect(result.exitCode).toBe(0)  // grep returns 0 if pattern found
})

// BETTER: Tests actual behavior via e2e tests
test('logout works end-to-end', async () => {
  await sandbox.exec('npm run build')
  const result = await sandbox.exec('npm run test:e2e -- --grep "logout"')
  expect(result.exitCode).toBe(0)
})
```

---

## Designing Effective Evals

Writing good evals is as important as writing good tests. Here's what makes an eval effective:

**Keep evals focused:** Each eval should test ONE capability. "Add a logout button" is better than "Add logout, refactor auth, and add tests."

**Target 1-5 minute completion:** If an eval consistently takes 10+ minutes, the task may be too complex. Break it into smaller evals.

**Test observable outcomes, not implementation:** Check that the button exists and works, not that the agent used a specific CSS class or variable name.

**Include negative tests:** Verify the agent didn't break existing functionality:
```typescript
test('existing tests still pass', async () => {
  const result = await sandbox.exec('npm test')
  expect(result.exitCode).toBe(0)
})
```

**Avoid ambiguous success criteria:** "Make it look better" is untestable. "Center the header text and use 16px font size" is testable.

**Start simple, then add complexity:** Get a basic eval working before adding edge cases and additional requirements.

---

## How Scoring Works

Evals are **pass or fail**—there's no partial credit.

**Execution order** (each step must succeed before the next runs):

```
1. Setup function (if defined)
       ↓
2. Agent executes the prompt
       ↓
3. npm scripts from `scripts` array, in order
   (stops on first failure)
       ↓
4. All vitest tests in EVAL.ts
       ↓
   PASS or FAIL
```

If any step fails, the entire eval run is marked as failed and subsequent steps are skipped.

### Failure Modes

Understanding why an eval failed helps you debug effectively:

| Failure Type | What Happened | How to Debug |
|--------------|---------------|--------------|
| **Agent timeout** | Agent exceeded the `timeout` limit (default: 5 min) | Check `transcript.jsonl` for loops or stuck states |
| **Agent crash** | Agent encountered an unrecoverable error | Check `transcript.jsonl` for error messages |
| **Script failure** | An npm script (build, lint, etc.) returned non-zero | Check `outputs/build.txt` or relevant script log |
| **Test failure** | EVAL.ts tests failed assertions | Check `outputs/tests.txt` for assertion details |
| **No changes** | Agent completed but didn't modify any files | Tests run on unchanged code; likely fails |
| **Setup failure** | The `setup` function threw an error | Check error output; verify sandbox access |

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

### How much does running evals cost?

Each eval run incurs two types of costs:

| Cost Type | Source | Rough Estimate |
|-----------|--------|----------------|
| **AI API costs** | Anthropic API usage | Depends on model and tokens used |
| **Compute costs** | Vercel Sandbox runtime | Billed per minute of compute time |

**Rough cost per run (1-5 minute eval):**
- Haiku: ~$0.10-$0.50
- Sonnet: ~$0.25-$1.00
- Opus: ~$0.50-$2.00

**Example:** Running 10 evals × 10 runs = 100 runs could cost $50-$200 with Opus.

*Note: These are rough estimates. Check [Anthropic's pricing](https://anthropic.com/pricing) and Vercel billing for current rates.*

**Cost-saving tips:**
- Use `model: 'haiku'` while developing and debugging evals
- Start with `runs: 1` until your eval is working correctly
- Switch to `model: 'opus'` and higher `runs` only for final measurements
- Use `earlyExit: true` (default) if you only need to know "does it work at all?"

---

## Troubleshooting

### Quick Reference

| Problem | Solution |
|---------|----------|
| "Why did my eval fail?" | Check `transcript.jsonl` to see what the agent did |
| "Agent seemed to succeed but tests failed" | Check `outputs/tests.txt` for test error details |
| "Setup keeps failing" | Verify your eval fixture has a valid `package.json` |
| "Missing VERCEL_TOKEN" | Add `VERCEL_TOKEN` or `VERCEL_OIDC_TOKEN` to your `.env` file |
| "Eval timed out" | Increase `timeout` in config, or simplify the task |

### Common Errors and What They Mean

#### Authentication Errors

```
Error: Invalid API key
```
**Cause:** Your `ANTHROPIC_API_KEY` is missing, expired, or incorrect.
**Fix:** Check your `.env` file and verify the key at [console.anthropic.com](https://console.anthropic.com).

```
Error: VERCEL_TOKEN is required
```
**Cause:** Missing Vercel credentials for sandbox access.
**Fix:** Add `VERCEL_TOKEN=your-token` to your `.env` file. Get a token from your [Vercel account settings](https://vercel.com/account/tokens).

#### Rate Limiting

```
Error: Rate limit exceeded (429)
```
**Cause:** You've made too many API requests in a short time.
**Fix:** Wait a few minutes, then retry. Consider reducing `runs` count or adding delays between experiments.

#### Timeout Errors

```
Error: Agent timed out after 300s
```
**Cause:** The agent took too long to complete the task.
**Fix:**
- Check if your task is too complex—consider breaking it into smaller evals
- Check `transcript.jsonl` to see if the agent got stuck in a loop
- Verify your eval fixture doesn't have dependency installation issues

#### EVAL.ts Syntax Errors

```
Error: Failed to parse EVAL.ts
SyntaxError: Unexpected token at line 15
```
**Cause:** Your test file has a JavaScript/TypeScript syntax error.
**Fix:** Run `npx tsc EVAL.ts --noEmit` locally to check for syntax errors before running the eval.

#### Sandbox Errors

```
Error: Sandbox creation failed
```
**Cause:** Issue provisioning the isolated VM.
**Fix:**
- Check your network connection
- Verify your Vercel token has the correct permissions
- Try again—this can be a transient infrastructure issue

```
Error: Command failed: npm install (exit code 1)
```
**Cause:** Dependencies failed to install in the sandbox.
**Fix:**
- Check that your `package.json` has valid dependencies
- Verify all packages are published to npm (no private/local packages)
- Check `outputs/install.txt` for detailed error logs

#### Test Failures

```
FAIL  EVAL.ts > logout button exists in codebase
AssertionError: No logout button found
```
**Cause:** Your test ran, but the assertion failed—the agent didn't complete the task correctly.
**Fix:**
- Check `transcript.jsonl` to see what the agent actually did
- Your prompt might be ambiguous—try making it more specific
- The task might be too hard for the model—try a more capable model

#### EVAL.ts Runtime Errors

```
Error: Cannot read properties of undefined (reading 'includes')
```
**Cause:** Your test code threw an exception—often from accessing properties on undefined values.
**Fix:** Common causes:
- Forgetting to `await` an async function (variable is a Promise, not the result)
- Assuming a file exists when it doesn't—use try/catch around `sandbox.readFile()`
- Parsing command output without checking `exitCode` first

### Debug Checklist

When an eval fails unexpectedly, check these files in order:

1. **`result.json`** — Did it pass or fail? What was the exit code?
2. **`transcript.jsonl`** — What did the agent do? Did it understand the task?
3. **`outputs/install.txt`** — Did dependencies install correctly?
4. **`outputs/build.txt`** — Did the build succeed?
5. **`outputs/tests.txt`** — What did the test output say?

### Still Stuck?

If you've checked all of the above and still can't figure out the issue:

1. Try running with `runs: 1` to isolate the problem
2. Simplify your EVAL.ts to a single basic test
3. Check that your eval works with a simpler prompt first
4. Verify your fixture runs correctly outside the framework (`npm install && npm run build`)
