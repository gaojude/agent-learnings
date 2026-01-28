# Forcing LLMs to Read Documentation Before Acting

## Summary
How to craft a prompt instruction that makes LLMs actually read docs instead of relying on stale pre-training knowledge.

## Context
When using Claude Code with Next.js documentation (via `npx @judegao/next-agents-md`), the agent would ignore the docs and use outdated pre-training knowledge. For example, creating `middleware.ts` instead of the new Next.js 16 `proxy.ts` convention.

## The Problem
The original instruction in CLAUDE.md was:
```
IMPORTANT: Prefer retrieval-led reasoning over pre-training-led reasoning for any Next.js tasks.
```

This **did not work** (0% pass rate) because:
1. Agent has high confidence in its pre-training knowledge
2. "Prefer" is too weak - doesn't create doubt
3. No actionable steps for the agent to follow
4. Agent doesn't feel uncertain, so doesn't look things up

## The Journey

### Attempt 1: "CRITICAL" + "MUST"
```
CRITICAL: You MUST read the relevant documentation files listed above BEFORE writing any Next.js code.
```
**Result: ~20-40% pass rate** - Better but inconsistent.

### Attempt 2: Adding "WRONG"
```
STOP. Next.js 16 has breaking changes. What you remember about Next.js is likely WRONG. Before writing ANY Next.js code, search the docs index and read the relevant section first. Your pre-training is outdated.
```
**Result: 100% (3/3)** - The word "WRONG" creates doubt!

### Attempt 3: Shortening (too aggressive)
```
What you remember about Next.js is WRONG. Always read docs first.
```
**Result: 66% (2/3)** - Too short, lost effectiveness.

### Attempt 4: Version-agnostic + "this project"
```
STOP. What you remember about Next.js is WRONG for this project. Always search docs and read before any task.
```
**Result: 100% (6/6)** - Winner!

## The Winning Prompt
```
STOP. What you remember about Next.js is WRONG for this project. Always search docs and read before any task.
```

**Key elements:**
1. **"STOP"** - Attention grabber, interrupts default behavior
2. **"WRONG for this project"** - Creates doubt without claiming universal wrongness
3. **"Always search docs and read"** - Actionable instruction
4. **"before any task"** - Applies to everything, not just code writing

## Why It Works
The phrase **"What you remember is WRONG"** is psychologically effective because:
- It creates **doubt** in the model's confidence
- It shifts from "prefer docs" (optional) to "your memory is wrong" (mandatory)
- The model can't trust its pre-training, so it MUST look things up

## Implementation
Used with the next-evals-oss CLI:
```bash
bun cli.ts --eval agent-039-indirect-proxy \
  --claude-code \
  --pre-hook "npx @judegao/next-agents-md --md CLAUDE.md" \
  --string-replace "IMPORTANT: Prefer retrieval-led reasoning over pre-training-led reasoning for any Next.js tasks.|STOP. What you remember about Next.js is WRONG for this project. Always search docs and read before any task." \
  --retries 0 --dry
```

## Results Comparison

| Instruction | Pass Rate |
|-------------|-----------|
| "Prefer retrieval-led reasoning" | 0% |
| "CRITICAL: MUST read docs" | ~20-40% |
| "What you remember is WRONG" (long) | 100% |
| "WRONG for this project" (short) | 100% |

## Full Eval Suite Results
With the winning prompt, running all 20 agent evals:
- **19/20 passed (95%)** without retries
- The new `agent-039-indirect-proxy` eval passed
- Only `agent-030-app-router-migration-hard` failed (unrelated React issue)

## Related
- `npx @judegao/next-agents-md` - Tool to inject Next.js docs into CLAUDE.md
- `--string-replace` CLI option in next-evals-oss
- Tags: #prompting #llm #documentation #next-evals-oss #retrieval
