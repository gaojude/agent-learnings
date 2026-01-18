# Skill Timing Matters: Docs Must Be Read BEFORE Code Exploration

## Summary
When designing skills that provide documentation, the timing of when docs enter the context matters more than whether they're read at all. Docs must shape the initial approach, not confirm/correct it after the fact.

## Context
Discovered while evaluating different approaches to providing Next.js documentation to Claude Code during coding tasks. This applies to any skill that provides reference documentation or knowledge that should inform implementation decisions.

## The Problem

Evaluated four approaches:
- **Baseline**: No docs (32/50 pass rate)
- **+CLAUDE.md**: Docs index embedded in CLAUDE.md before task starts (42/50 pass rate)
- **+SKILL.md**: On-demand skill invocation (36/50 pass rate)
- **+SKILL2**: Skill + CLAUDE.md nudge to invoke (36/50 pass rate)

**Why did CLAUDE.md (42/50) outperform SKILL2 (36/50) when both read docs?**

## Key Finding: Timing of Documentation Access Determines Success

### CLAUDE.md (winner): Docs shape initial problem framing
- Docs available in context BEFORE model starts reasoning
- Model reads docs as FIRST action
- Pattern: `[Read docs] → [Glob files] → [Read code] → [Write]`

### SKILL2 (original - worse): Docs arrive too late
- Model explores code first, then seeks docs
- Docs become confirmation/correction rather than formative guidance
- Pattern: `[Skill invoke] → [Read code] → [Read docs] → [Write]`

## The Critical Bug

**Contradictory instructions in original SKILL2:**
- CLAUDE.md said: "You MUST invoke the skill"
- SKILL.md said: "Use AFTER exploring the codebase first"

This caused the model to explore code FIRST, undermining the timing advantage.

## The Fix

### Improved SKILL.md
```markdown
---
name: nextjs-doc
description: "Next.js official documentation. Invoke FIRST before reading project code."
---

# Next.js Documentation

This skill provides official Next.js documentation matching the project's Next.js version.

Your training data may be outdated. Consult these docs for current APIs and patterns.

[doc index]
```

### Improved CLAUDE.md
```markdown
# Next.js Project

## CRITICAL: Read Documentation FIRST

Before reading ANY code files in this project, you MUST:

1. **Invoke the `nextjs-doc` skill** as your FIRST action
2. **Read the relevant doc files** for the task
3. **Only then** explore the existing codebase

Your training data may be outdated. The skill provides current documentation that should inform your approach from the start.
```

## Results After Fix

| Eval | Before | After |
|------|--------|-------|
| 029-use-cache-directive | ❌ | ✅ |
| 039-parallel-routes | ❌ | ✅ |

SKILL2 now matches CLAUDE.md performance on timing-sensitive evals.

## Key Principles

1. **Timing > Content**: When docs enter the context matters more than whether they're read at all
2. **Explicit numbered steps**: "1. Do X, 2. Do Y, 3. Do Z" removes ambiguity about order
3. **Remove contradictions**: Don't say "must invoke" in one place and "explore first" in another
4. **Proactive > Reactive**: Docs that shape initial approach outperform docs consulted after problems arise
5. **Context position matters**: Information at conversation start has higher attention weight than mid-conversation additions

## Transcript Analysis Evidence

Analyzed JSONL transcripts from `~/.claude/projects/` to confirm behavioral differences:

**CLAUDE.md first actions:**
1. Read `parallel-routes.mdx` (docs)
2. Glob files
3. Read code

**Original SKILL2 first actions:**
1. Skill invoke
2. Read code files
3. Read docs (too late)

**Fixed SKILL2 first actions:**
1. Skill invoke
2. Read `parallel-routes.mdx` (docs)
3. Glob files
4. Read code

## Code Location
- File: `lib/claude-code-runner.ts`
- Function: `createNextjsSkill2()`
- Project: `next-evals-oss`

## Related
- Claude Code skills design
- CLAUDE.md best practices
- Context window attention patterns
- Documentation-driven development
