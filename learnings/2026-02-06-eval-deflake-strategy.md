# Eval Deflake Strategy: Run, QA, Remove, Rerun

**Date:** 2026-02-06
**Project:** next-evals-oss (Next.js agent evals)

## The Problem

When running AI agent evals at scale (many models × many evals), failures come in three flavors:
- **Model failures** — the AI genuinely couldn't solve the task
- **Infra failures** — rate limiting, network errors, sandbox issues, CLI install failures
- **Timeout failures** — the eval exceeded its time limit (ambiguous cause)

Only model failures are meaningful signal. Infra and timeout failures are noise that must be eliminated before publishing results.

## The Strategy: Run → QA → Remove → Rerun (loop)

### 1. Run with deduplication
Run all (model, eval) pairs. The runner tracks which pairs have already completed (via results on disk) and only runs missing ones. This means you can safely re-run at any time — it picks up where it left off.

### 2. QA: classify and remove bad results
After a run completes, run a QA pass that:
- Classifies every failure as model/infra/timeout using an AI classifier
- **Deletes result folders** for any non-model failure (infra or timeout)
- Exports only clean results (successes + model failures)

Deleting the result folder is the key insight — it makes the runner think that pair was never attempted, so the next run will pick it up automatically.

### 3. Rerun to fill gaps
Run the eval runner again. It detects the missing pairs (the ones QA deleted) and re-runs only those. This is cheap because most results are already cached.

### 4. Loop until clean
Repeat steps 2-3 until the QA pass reports zero infra/timeout failures. At that point, every failure in the dataset is a genuine model failure.

## Why This Works

- **Convergent:** Each loop reduces noise. Transient infra issues resolve on retry. Persistent issues (genuine model failures) stabilize.
- **Idempotent:** Every step is safe to re-run. The runner deduplicates, QA is memoized (cached classification per eval), and deletion + rerun is the recovery mechanism.
- **No manual intervention:** The loop is fully automated. No need to manually inspect failures or decide what to retry.

## Inspiration for Upstream (@vercel/agent-eval)

This pattern could be built into agent-eval itself:
- `agent-eval run` already deduplicates — good
- Add `agent-eval qa` that classifies failures, deletes non-model results, and reports what needs rerunning
- Add `agent-eval run --until-clean` that loops run→qa→run until zero infra/timeout failures remain
- The classifier result should be cached (e.g., `classification.json` alongside `summary.json`) to avoid redundant AI calls
