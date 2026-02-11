# Running Agent Evals Against Next.js

**Date:** 2026-02-11

**For:** Anthropic team (or anyone without Vercel credentials)

**Repo:** `/Users/judegao/workspace/repo/next.js`

**Framework:** @vercel/agent-eval

---

## Quick Start

### 1. Prerequisites

Only need:
```bash
# Claude API key (free tier available)
ANTHROPIC_API_KEY=sk-ant-...

# Docker (for sandboxes - no Vercel account needed)
docker --version  # Must be installed
```

That's it. No Vercel token, no AI Gateway key needed.

### 2. Create Eval Project

```bash
# Initialize a new eval project
npx @vercel/agent-eval init next-evals
cd next-evals

# Install dependencies
npm install
```

### 3. Configure Environment

```bash
# Copy template
cp .env.example .env

# Edit .env - only fill in:
ANTHROPIC_API_KEY=sk-ant-...
# Leave VERCEL_TOKEN empty

# The framework will auto-detect Docker and use it
```

### 4. Create Your First Eval

Create `evals/app-router-migration/PROMPT.md`:
```markdown
# Migrate a Next.js Pages Router app to App Router

The user has a simple Next.js 13 app using the Pages Router.

## Task
Convert `pages/index.js` to the App Router structure:
- Create `app/page.js` with equivalent functionality
- Update any imports to use Next.js 13+ APIs
- Ensure `getStaticProps` logic is migrated to `generateStaticParams` if needed

## Current Code
The app is in the `/scaffold` directory.

## Success Criteria
- App Router structure exists (`app/page.js`)
- Page renders without errors
- Build succeeds with `npm run build`
```

Create `evals/app-router-migration/EVAL.ts`:
```typescript
import { test, expect } from 'vitest';
import { readFileSync, existsSync } from 'fs';
import { execSync } from 'child_process';

test('app/page.js exists', () => {
  expect(existsSync('app/page.js')).toBe(true);
});

test('app/page.js is valid JSX/TSX', () => {
  const content = readFileSync('app/page.js', 'utf-8');
  expect(content).toContain('export default');
});

test('project builds', () => {
  execSync('npm run build', { stdio: 'pipe' });
});
```

Create `evals/app-router-migration/package.json`:
```json
{
  "name": "next-app-router-migration",
  "version": "1.0.0",
  "dependencies": {
    "next": "^15.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "scripts": {
    "build": "next build",
    "dev": "next dev"
  }
}
```

Create `evals/app-router-migration/src/pages/index.js`:
```javascript
// TODO: This is the starting code that needs to be migrated to App Router
export default function Home() {
  return <h1>Welcome to Next.js</h1>;
}
```

Your directory structure:
```
next-evals/
├── .env
├── experiments/
│   └── next.ts            # (create this next)
├── evals/
│   └── app-router-migration/
│       ├── PROMPT.md
│       ├── EVAL.ts
│       ├── package.json
│       └── src/
│           └── pages/
│               └── index.js
└── results/               # (auto-created with results)
```

### 5. Create Experiment Config

Create `experiments/next.ts`:
```typescript
import type { ExperimentConfig } from '@vercel/agent-eval';

const config: ExperimentConfig = {
  agent: 'claude-code',       // Direct Anthropic API
  model: 'opus',              // or 'sonnet'
  runs: 3,
  earlyExit: false,           // Run all to see reliability
  timeout: 600,               // 10 minutes

  // Use Docker - no Vercel account needed
  sandbox: 'docker',

  // These npm scripts verify the output
  scripts: ['build'],

  // Optional: filter which evals to run
  evals: '*',
};

export default config;
```

### 6. Preview (No Cost)

```bash
# See what will run without making API calls
npx @vercel/agent-eval --dry

# Or just the next experiment
npx @vercel/agent-eval next --dry
```

### 7. Run Evals

```bash
# Smoke test first (runs 1 eval, 1 time)
npx @vercel/agent-eval next --smoke

# Once verified, run the full experiment
npx @vercel/agent-eval next
```

### 8. View Results

```bash
# Open web-based results viewer
npx @vercel/agent-eval playground
```

Results are also saved to `results/next/<timestamp>/` as JSON.

---

## Understanding Results

### Results Directory Structure
```
results/next/2026-02-11T10-30-00Z/
├── app-router-migration/
│   ├── summary.json         # Pass rate, fingerprint
│   ├── run-1/
│   │   ├── result.json      # Status, duration, error
│   │   ├── transcript.json  # Agent actions & thinking
│   │   ├── outputs/
│   │   │   ├── eval.txt     # Test output
│   │   │   └── scripts/
│   │   │       └── build.txt
│   │   └── ...
│   ├── run-2/
│   └── run-3/
```

### Summary JSON Example
```json
{
  "totalRuns": 3,
  "passedRuns": 2,
  "passRate": "67%",
  "meanDuration": 45.2,
  "fingerprint": "a1b2c3..."
}
```

### What Each Column Means

| Field | Meaning |
|-------|---------|
| `passedRuns` | How many runs passed all tests |
| `passRate` | Success percentage across runs |
| `meanDuration` | Average seconds per run |
| `fingerprint` | Hash of eval config—used to reuse results |

---

## Tips

### 1. Start Simple
Begin with small migrations (Pages → App Router), not complex features.

### 2. Provide Context in PROMPT.md
- Show the current code structure
- Explain what APIs changed
- Give examples of the target output

### 3. Use Multiple Runs
Set `runs: 10` and `earlyExit: false` to measure reliability:
- 10/10 = reliable
- 7/10 = inconsistent
- 2/10 = model struggles with this task

### 4. Iterate on Prompts
If pass rate is low:
1. Read `transcript.json` to see what the agent tried
2. Check `eval.txt` to see why tests failed
3. Improve the PROMPT.md with:
   - Clearer instructions
   - More examples
   - Explicit requirements

### 5. Compare Approaches
Create multiple experiments to compare:
```typescript
// experiments/baseline.ts - default
// experiments/with-examples.ts - with code samples
// experiments/sonnet-vs-opus.ts - model comparison
```

Then compare `passRate` across results.

---

## Troubleshooting

### Docker Not Found
```bash
# Install Docker Desktop from https://www.docker.com/products/docker-desktop
# Or use Docker via Colima on macOS

# Verify installation
docker ps
```

### Agent Produces Nothing
- Check `transcript.json` — did Claude Code initialize?
- Check ANTHROPIC_API_KEY is set correctly
- Run with `--smoke` first to test setup

### Build Fails in EVAL
- The eval sandbox is isolated from your Next.js repo
- Make sure `evals/*/src` has all needed files
- Include `package.json` with required dependencies

### Tests Fail But Agent Produced Code
- Check `outputs/eval.txt` to see test output
- Update EVAL.ts to match what the agent produces
- Read `transcript.json` to see what agent intended

---

## Next Steps

### 1. Test Real Next.js Migrations
Create evals for actual Next.js migration paths:
- Pages Router → App Router
- Old `getStaticProps` → `generateStaticParams`
- Image component usage
- Font optimization

### 2. Test Feature Implementation
- Build a new Next.js component
- Add API routes
- Set up middleware

### 3. Measure Across Models
```typescript
model: ['opus', 'sonnet', 'haiku']  // Compare models
```

### 4. A/B Test Documentation
```typescript
// experiments/good-docs.ts
// experiments/minimal-docs.ts
// Compare pass rates to measure doc impact
```

---

## Reference: Full Example Config

```typescript
// experiments/next-migrations.ts
import type { ExperimentConfig } from '@vercel/agent-eval';

const config: ExperimentConfig = {
  agent: 'claude-code',
  model: 'opus',
  runs: 5,
  earlyExit: false,        // Run all 5 times
  timeout: 900,            // 15 minutes for complex tasks
  sandbox: 'docker',       // No Vercel token needed

  scripts: ['build', 'lint'],  // Optional: verify build & lint pass

  // Setup can customize the sandbox before agent starts
  setup: async (sandbox) => {
    // Install additional tools if needed
    await sandbox.runCommand('npm', ['install', '-g', 'pnpm']);
  },

  // Filter specific evals
  evals: (name) => name.includes('app-router'),
};

export default config;
```

---

## No Classifier Warning

You'll see this warning when running:
```
⚠️  Classifier disabled: Neither AI_GATEWAY_API_KEY nor VERCEL_OIDC_TOKEN is set.
  The classifier automatically identifies why evals failed...
  Set AI_GATEWAY_API_KEY or VERCEL_OIDC_TOKEN to enable classifier...
```

**This is fine!** It just means:
- ✅ Evals still run normally
- ✅ Results are saved
- ✅ Tests run and report pass/fail
- ❌ The framework won't auto-classify why failures happened (model vs infra)

You can still read transcripts manually to understand failures.

---

## Questions?

Refer back to the main @vercel/agent-eval README for:
- A/B testing patterns
- Result reuse mechanism
- More complex configurations
- Playground UI guide
