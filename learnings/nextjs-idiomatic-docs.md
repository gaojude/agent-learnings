# Next.js Eval: Idiomatic + Docs Synergy

**Date:** 2026-01-16
**Project:** next-evals-oss
**Test case:** `000-app-router-migration-simple`

## Finding

Neither "idiomatic" keyword alone nor docs alone reliably produce idiomatic code. But together they work - the keyword primes Claude to look for best practices, and the docs provide the specific knowledge (e.g., Link component).

## Results

| Prompt | Claude Code | CC + Docs |
|--------|-------------|-----------|
| Without "idiomatic" | fail | fail |
| With "idiomatic" | fail | pass |

## Implication

Prompt wording matters for getting full value from documentation context. When you want Claude to apply best practices from docs, use words like "idiomatic" to prime the behavior.
