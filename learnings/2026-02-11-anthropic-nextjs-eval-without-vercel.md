# Anthropic Can Run Next.js Eval Without Vercel Credentials

**Date:** 2026-02-11

**Project:** @vercel/agent-eval

## Context

Made the classifier feature optional in @vercel/agent-eval framework. This unblocks Anthropic and other organizations that don't have Vercel accounts from using the framework.

## Key Changes

### Classifier Made Optional
- Created `isClassifierEnabled()` feature flag that checks for `AI_GATEWAY_API_KEY` or `VERCEL_OIDC_TOKEN`
- If neither env var is set, classifier is completely skipped
- No hard requirement for AI_GATEWAY_API_KEY anymore

### Housekeeping Behavior
- Non-model failures only cleaned up when classifier is enabled
- When disabled, housekeeping preserves all results (only removes incomplete/duplicates)
- Prevents unexpected deletion of results that couldn't be classified

### Warnings & Documentation
- CLI displays warning when classifier is disabled, explaining why keys are needed
- Updated README with clear "Direct API keys" section
- Updated init template to show all options

## Usage for Anthropic

### Minimal Setup
```bash
# Only need Claude API key
ANTHROPIC_API_KEY=sk-ant-...
```

### Experiment Config
```typescript
const config: ExperimentConfig = {
  agent: 'claude-code',      // Direct API (not vercel-ai-gateway/...)
  model: 'opus',
  runs: 1,
  sandbox: 'docker',         // No Vercel token needed!
};
```

### What Works
- ✅ Run evals with Claude Code
- ✅ Use Docker sandbox (no Vercel account needed)
- ✅ View all results and transcripts
- ✅ Use all agent tools and features
- ❌ Auto-classify failures (displays warning but continues)
- ❌ Auto-remove infra/timeout failures (can be done manually)

## Implementation Details

**Changed Files:**
- `src/lib/classifier.ts` — Added `isClassifierEnabled()` function
- `src/cli.ts` — Conditional classification block + warning message
- `src/lib/housekeeping.ts` — Conditional non-model failure cleanup
- `README.md` — New "Direct API keys" guide section
- `src/lib/init.ts` — Updated env template and README template

**Tests Added:**
- Test for housekeeping behavior when classifier is disabled
- All existing tests still pass

## Message to Share

"The framework now works with just `ANTHROPIC_API_KEY` + Docker. The classifier (which requires Vercel keys) is optional. You'll get a warning that it's disabled, but all features work fine—evals run, results are saved, everything's there."
