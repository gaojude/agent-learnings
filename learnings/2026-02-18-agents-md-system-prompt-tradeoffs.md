# AGENTS.md System Prompt Tradeoffs

**Date:** 2026-02-18
**Project:** next-evals-oss (Next.js agent evaluation framework)

## Key Finding

Adding documentation (AGENTS.md) to the system prompt consistently improves AI coding agent performance, but it's a tradeoff — not a free win.

## Data

Tested across 7 models (Claude Opus 4.6, Claude Sonnet 4.5, Cursor Composer 1.5, Gemini 3.0 Pro Preview x2, GPT 5.2 Codex, GPT 5.3 Codex) with 20 Next.js coding evals each:

- **Improvement range:** +10% to +30% across all models
- **Regressions:** 0 newly failed evals across 140 eval pairs
- **Two models hit 100%** with docs (Claude Opus 4.6, GPT 5.3 Codex)

## The Tradeoff

More context in the system prompt steers models toward correct patterns, but can cause over-application:

- Example: Claude Opus 4.6 read cache docs and added `'use cache'` to code it wasn't asked to touch, breaking the build
- The model was "helpfully" modernizing existing code based on what it learned from the docs
- The same docs that helped it pass 5 new evals caused it to over-correct on another

## Realistic Expectations

- **100% is unrealistic.** Adding instructions helps the average case but can interfere with edge cases.
- **The right bar:** Does it make the average case significantly better without making any individual case significantly worse?
- **Zero regressions is achievable** but shouldn't be expected as the norm — it's the best-case outcome.
- **"How much is too much?"** is hard to answer in the abstract. But zero regressions across 140 eval pairs is about as clean as you can hope for.
