# Agent Eval vs LLM Eval for Coding Tasks

## Summary
For realistic coding evals (like Next.js migrations), use agent-based evaluation instead of single-shot LLM evaluation.

## Context
When building an eval framework to test AI coding capabilities on tasks like Next.js App Router migrations, server/client components, and multi-file refactoring.

## Details

### Why Agent Eval is Better for Coding Tasks

| Aspect | LLM Eval (single-shot) | Agent Eval |
|--------|------------------------|------------|
| **Can read existing files** | Only what you paste in prompt | Explores codebase |
| **Can iterate on errors** | One attempt | Sees build/lint errors, fixes them |
| **Can run commands** | No | Yes (npm, git, etc.) |
| **Realistic workflow** | Artificial | Matches how devs actually use AI |
| **Multi-file changes** | Must output all at once | Can make incremental changes |
| **Context management** | You manage it | Agent decides what to read |

### For Next.js Evals Specifically

Tasks like these require agent capabilities:
- App Router migration - needs to understand existing structure
- Server/Client components - requires reading multiple files
- Parallel routes, intercepting routes - complex multi-file setup
- Any task where the agent needs to iterate until build passes

### Recommendation for next-evals-oss

1. **Remove LLM mode** from `cli.ts` - it's ~500 lines of code that produces inferior results
2. **Keep agent runners** (Claude Code, Gemini, Codex, Cursor, Copilot, Opencode)
3. **Keep LLM Judge** for scoring - single-shot eval is appropriate for classification/scoring tasks

### When Single-Shot LLM Eval IS Appropriate

- Code completion benchmarks (HumanEval-style)
- Classification tasks
- Simple transformations with full context provided in prompt
- Scoring/judging agent outputs (like LLM Judge)

## Related
- Tags: #evals #agents #llm #next-js #coding-benchmarks
- Project: next-evals-oss
