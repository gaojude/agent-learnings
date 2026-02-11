# Running next-evals-oss Without Vercel Credentials

**Date:** 2026-02-11

**Project:** `next-evals-oss` (existing eval suite at `/Users/judegao/workspace/repo/next-evals-oss`)

**For:** Anthropic, or anyone without Vercel API Gateway / OIDC tokens

---

## Overview

The `next-evals-oss` project has **40+ production Next.js evals** already set up. Thanks to recent changes in `@vercel/agent-eval`, you can now run all of them with **just `ANTHROPIC_API_KEY` and Docker** — no Vercel tokens needed.

**What you'll get:**
- ✅ Run all 40 evals against Claude Code (any model)
- ✅ Measure pass rates across multiple runs
- ✅ Compare different models (Opus, Sonnet, Haiku)
- ✅ Export results to JSON for analysis
- ✅ No Vercel account required
- ❌ Failure classification disabled (but still runs fine)

---

## Prerequisites

### Minimum Setup
```bash
# 1. Claude API key
ANTHROPIC_API_KEY=sk-ant-...

# 2. Docker installed
docker --version    # Should return version info
```

That's it!

### (Optional) Vercel Token
If you have a Vercel token, you can use it instead of Docker:
```bash
VERCEL_TOKEN=<your-token>
```

---

## Getting Started

### 1. Clone and Install

```bash
cd /Users/judegao/workspace/repo/next-evals-oss

# Install framework and eval dependencies
npm install
```

### 2. Configure Environment

The project needs an `.env` file. The current `.env.local` requires Vercel keys. Create a new `.env` for your setup:

```bash
# Create .env (this is git-ignored)
cat > .env << 'EOF'
# Claude Code API
ANTHROPIC_API_KEY=sk-ant-...

# Sandbox - pick ONE option:

# Option 1: Docker (recommended - no additional setup)
# Just set this, Docker will be auto-detected

# Option 2: Vercel (if you have a token)
# VERCEL_TOKEN=<your-vercel-token>

# That's it!
EOF
```

### 3. Verify Setup

```bash
# Dry run - no cost, no API calls
npm run eval:dry

# You should see:
# - 40 evals listed
# - Models from experiments/ listed
# - "Would run X eval pairs"
```

---

## Running Evals

### Option 1: Quick Test (Recommended First)

```bash
# Smoke test - runs 1 eval per model, minimal cost
npm run eval:smoke

# Expected output:
# ✓ Running agent-000 (Pages Router → App Router) [1/1]...
# ✗ Running agent-021 (Avoid fetch in useEffect) [1/1]...
# (depending on model capability)
```

### Option 2: Run All Evals

```bash
# Runs all 40 evals for each model in experiments/
# Respects fingerprinting - only runs new/changed evals
npm run eval

# Or force re-run everything:
npm run eval -- --force
```

### Option 3: Run Specific Subset

Edit `experiments/claude-code.ts` to filter evals:

```typescript
// experiments/claude-code.ts
import type { ExperimentConfig } from '@vercel/agent-eval';

const config: ExperimentConfig = {
  agent: 'claude-code',
  model: 'opus',
  runs: 3,
  sandbox: 'docker',
  evals: (name) => name.includes('app-router') || name.includes('server'),
};

export default config;
```

Then:
```bash
npm run eval
```

---

## Understanding the Project Structure

### Top Level

```
next-evals-oss/
├── experiments/          # Config for each model
│   ├── claude-code.ts    # Direct API (no Vercel keys!)
│   ├── opus.ts           # (old Vercel config - skip this)
│   └── ...
├── evals/               # 40 Next.js tasks
│   ├── agent-000-pages-to-app-router/
│   ├── agent-021-avoid-fetch-useeffect/
│   ├── agent-031-proxy-middleware/
│   └── ... (agent-032 through agent-039)
├── results/             # Experiment results (auto-created)
├── scripts/
│   └── export-results.ts
├── npm run eval         # Main script
└── .env                 # Your configuration (git-ignored)
```

### Experiment Configs You Can Use

#### `experiments/claude-code.ts` (For Direct API)
```typescript
import type { ExperimentConfig } from '@vercel/agent-eval';

const config: ExperimentConfig = {
  agent: 'claude-code',
  model: 'opus',         // or 'sonnet', 'haiku'
  runs: 3,               // 3 runs per eval
  earlyExit: false,      // Run all to measure reliability
  sandbox: 'docker',     // No Vercel token needed
  timeout: 600,
};

export default config;
```

**Pro tip:** Create multiple experiment configs to compare:

```typescript
// experiments/claude-code-opus.ts
export default { agent: 'claude-code', model: 'opus', ... };

// experiments/claude-code-sonnet.ts
export default { agent: 'claude-code', model: 'sonnet', ... };
```

Then run:
```bash
npm run eval                    # Runs all experiment configs
```

### What Each Eval Tests

| Eval | Purpose | Difficulty |
|------|---------|------------|
| `agent-000` | Pages Router → App Router (basic) | Easy |
| `agent-021` | Remove async operations from client | Medium |
| `agent-022` | Convert API calls to server actions | Medium |
| `agent-023` | Remove getServerSideProps | Easy |
| `agent-024` | Fix redundant useState | Medium |
| `agent-025` | Use Next.js Link component | Easy |
| `agent-026` | Remove serial awaits | Hard |
| `agent-027` | Use Next.js Image | Medium |
| `agent-028` | Use Next.js Font | Easy |
| `agent-029` | Add cache directives | Medium |
| `agent-030` | Pages Router → App Router (advanced) | Hard |
| `agent-031` | Implement proxy (middleware) | Hard |
| ... | More advanced Next.js patterns | Varies |

---

## Viewing Results

### Real-time During Runs

```bash
# In another terminal, watch results appear live
npm run eval
# Results stream to console + saved to results/
```

### After Run Complete

#### Option 1: JSON Export

```bash
# Export to agent-results.json
npm run export-results

# View the JSON
cat agent-results.json | jq '.[] | {model, passRate}'

# Output:
# {
#   "model": "claude-opus",
#   "passRate": "72%"
# }
```

#### Option 2: Web Dashboard (if available)

```bash
# If you can access @vercel/agent-eval playground:
npx @vercel/agent-eval playground
```

#### Option 3: Raw Results Directory

```bash
# Results saved per experiment:
ls -la results/

# Example structure:
results/
├── claude-opus/
│   └── 2026-02-11T10-30-00Z/
│       ├── agent-000/
│       │   ├── summary.json
│       │   ├── run-1/
│       │   │   ├── result.json
│       │   │   ├── transcript.json    # Agent actions
│       │   │   └── outputs/
│       │   ├── run-2/
│       │   └── run-3/
│       ├── agent-021/
│       └── ...
```

### Reading Summary JSON

```bash
# See overall results for an eval:
cat results/claude-opus/2026-02-11T10-30-00Z/agent-000/summary.json

# Output:
{
  "totalRuns": 3,
  "passedRuns": 2,
  "passRate": "67%",
  "meanDuration": 45.2,
  "fingerprint": "abc123..."
}
```

### Debugging Failed Evals

When an eval fails, investigate:

```bash
# 1. Read the test output
cat results/claude-opus/2026-02-11T.../agent-000/run-1/outputs/eval.txt

# 2. See what agent produced
cat results/claude-opus/2026-02-11T.../agent-000/run-1/result.json

# 3. Read the full transcript (agent thinking + actions)
cat results/claude-opus/2026-02-11T.../agent-000/run-1/transcript.json | jq .
```

---

## Workflow: Running & Comparing Models

### 1. Set Up Multiple Configs

Create experiment files for each model:

```bash
# experiments/claude-code-opus.ts
const config = { agent: 'claude-code', model: 'opus', runs: 5, sandbox: 'docker' };
export default config;

# experiments/claude-code-sonnet.ts
const config = { agent: 'claude-code', model: 'sonnet', runs: 5, sandbox: 'docker' };
export default config;

# experiments/claude-code-haiku.ts
const config = { agent: 'claude-code', model: 'haiku', runs: 5, sandbox: 'docker' };
export default config;
```

### 2. Run All

```bash
npm run eval              # Runs all three model configs
npm run eval -- --force   # Force re-run everything
```

### 3. Compare Results

```bash
npm run export-results

# Results in agent-results.json:
{
  "claude-opus": { "passRate": "72%", "evals": [...] },
  "claude-sonnet": { "passRate": "68%", "evals": [...] },
  "claude-haiku": { "passRate": "45%", "evals": [...] }
}
```

---

## Important: Classifier Disabled Warning

When you run evals, you'll see:

```
⚠️  Classifier disabled: Neither AI_GATEWAY_API_KEY nor VERCEL_OIDC_TOKEN is set.
  The classifier automatically identifies why evals failed...
  Without it, all failed results are kept as-is...
```

**What this means:**

| Feature | Status |
|---------|--------|
| Running evals | ✅ Works |
| Tests pass/fail | ✅ Works |
| Results saved | ✅ Works |
| Auto-classify failures | ❌ Skipped |
| Auto-remove infra failures | ❌ Skipped |
| Export to JSON | ✅ Works |

**You can ignore this warning.** All evals run normally. You just won't have automatic classification of *why* failures happened (e.g., model error vs timeout).

---

## Troubleshooting

### Issue: "Docker not found"

```
Error: Docker is required but not installed
```

**Fix:**
1. Install Docker Desktop: https://www.docker.com/products/docker-desktop
2. Start Docker
3. Verify: `docker ps` should work

### Issue: "ANTHROPIC_API_KEY not set"

```
Error: ANTHROPIC_API_KEY environment variable not found
```

**Fix:**
```bash
echo "ANTHROPIC_API_KEY=sk-ant-..." > .env
source .env
npm run eval:dry  # Verify
```

### Issue: "Eval hangs or times out"

Docker sandbox may be slow. Increase timeout:

```typescript
// experiments/claude-code.ts
const config = {
  agent: 'claude-code',
  model: 'opus',
  timeout: 900,   // 15 minutes instead of 10
  sandbox: 'docker',
};

export default config;
```

### Issue: "Some evals already run, want to skip them"

The framework uses fingerprinting to skip completed evals. To re-run:

```bash
npm run eval                    # Skip already-run evals
npm run eval -- --force         # Re-run everything
```

### Issue: "Want to run just one eval"

Edit the experiment config to filter:

```typescript
// experiments/claude-code.ts
const config = {
  ...
  evals: ['agent-000'],  // Only this eval
};
```

Or:

```typescript
evals: (name) => name.startsWith('agent-03'),  // Only agent-030+
```

---

## Tips

### 1. Start with Smoke Test
```bash
npm run eval:smoke
# Runs 1 eval per model - fast feedback
```

### 2. Measure Reliability
Set `runs: 5` to measure how consistent the model is:
- 5/5 = very reliable
- 3/5 = inconsistent
- 0/5 = can't do this task

### 3. Compare Models Systematically

Create separate experiment configs and export results:
```bash
npm run export-results
# agent-results.json now has model-by-model breakdown
```

### 4. Read Transcripts When Confused

When results don't match expectations, always check:
```bash
cat results/<model>/<timestamp>/<eval>/run-1/transcript.json | jq '.events[] | {type, content}'
```

### 5. Iterate on Evals

If an eval has low pass rate:
1. Read the transcript — see what agent tried
2. Improve the PROMPT.md in `evals/<eval>/`
3. Re-run with `--force`

---

## Publishing Results (Optional)

If you want to contribute results back to nextjs.org:

```bash
npm run export-results
# Creates/updates agent-results.json

# Copy to Next.js front repo:
cp agent-results.json <path-to-front>/apps/next-site/app/\(next-site\)/evals/
```

---

## File Structure Reference

### Experiment Config Format

```typescript
// experiments/my-config.ts
import type { ExperimentConfig } from '@vercel/agent-eval';

const config: ExperimentConfig = {
  agent: 'claude-code',          // Must use direct API (not vercel-ai-gateway)
  model: 'opus',                 // 'opus' | 'sonnet' | 'haiku'
  runs: 3,                        // How many times per eval
  earlyExit: false,              // Run all runs even after first pass
  sandbox: 'docker',             // No other options needed
  timeout: 600,                  // Seconds per run
};

export default config;
```

### Result Summary Format

```json
{
  "totalRuns": 3,
  "passedRuns": 2,
  "passRate": "67%",
  "meanDuration": 45.2,
  "fingerprint": "abc123...",
  "model": "opus"
}
```

---

## Next Steps

1. **Run smoke test** — `npm run eval:smoke`
2. **Pick a model** — Edit `experiments/claude-code.ts`
3. **Run all evals** — `npm run eval`
4. **Export results** — `npm run export-results`
5. **Analyze** — Compare pass rates across evals

**Question?** Check transcript.json files — they show exactly what the agent tried and why it succeeded or failed.
