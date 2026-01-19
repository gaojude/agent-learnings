# Why +CLAUDE.md Outperforms Skill-Based Documentation Delivery

## Summary
Investigation into why embedding docs in CLAUDE.md consistently outperforms skill-based doc delivery revealed that **exploration order is the key factor** - when Claude invokes a skill first, it creates a "knowledge anchor" that prevents holistic task completion.

## Context
Eval `029-use-cache-directive` showed consistent pattern: +CLAUDE.md passed while +SKILL.md and SKILL2 failed. This triggered a deep investigation into transcript analysis to understand root cause.

## The Investigation

### Step 1: Observe the Pattern
Running `--compare-nextjs-docs` showed:
- Baseline: ❌
- +CLAUDE.md: ✅
- +SKILL.md: ❌
- SKILL2: ❌ (sometimes)

### Step 2: Transcript Analysis
Analyzed Claude's conversation transcripts to trace file access order:

**+CLAUDE.md (SUCCESS):**
```
next.config.ts → page.tsx → db.js → [discovers docs in temp folder] → Edit next.config.ts → Write page.tsx
```

**SKILL approach (FAILURE):**
```
[Invoke skill] → [Read docs] → page.tsx → db.js → Write page.tsx
→ NEVER touched next.config.ts!
```

### Step 3: Form Hypothesis
**Knowledge Anchor Hypothesis**: When Claude invokes a skill and reads documentation FIRST, it:
1. Anchors on the patterns shown in docs
2. Applies those patterns immediately
3. Never explores the project to discover OTHER requirements (like config changes)

The `use cache` directive requires BOTH:
- `experimental: { cacheComponents: true }` in next.config.ts
- Proper `'use cache'` usage in page.tsx

SKILL approach only did page.tsx because that's what the docs emphasized.

### Step 4: Verify with File Access Patterns
Confirmed by checking transcript tool calls:
- SKILL2 failure: 0 file operations on next.config.ts
- +CLAUDE.md success: 2 file operations on next.config.ts (read + edit)

### Step 5: Test the Hypothesis
Added "explore first" nudge to SKILL2:
```
Before writing code, first explore the project structure, then invoke the `nextjs-doc` skill for documentation.
```

**Result: SKILL2 now passes** - matching +CLAUDE.md performance.

## Root Cause

The key difference is **when documentation enters Claude's context**:

| Approach | Doc Discovery | Mental Model | Result |
|----------|---------------|--------------|--------|
| +CLAUDE.md | During natural exploration | Holistic (project + docs together) | ✅ |
| SKILL (no nudge) | Before exploration | Anchored on doc patterns only | ❌ |
| SKILL + explore-first | After exploration | Holistic (project context first) | ✅ |

## Why +CLAUDE.md Always Works

Embedding docs in CLAUDE.md or temp folders creates **integrated discovery**:
1. Claude explores project naturally
2. Discovers docs as part of exploration
3. Applies docs WITH full project context
4. No cognitive separation between "learning" and "doing"

## Key Takeaway

**Exploration order determines task success.** When Claude reads documentation before understanding the project, it applies docs literally without considering project-specific requirements. The fix is ensuring project exploration happens BEFORE or ALONGSIDE documentation consumption.

## Related
- Eval: `029-use-cache-directive`
- Verification method: Transcript analysis (`~/.claude/projects/{path}/{uuid}.jsonl`)
- Tags: skills, CLAUDE.md, exploration-order, knowledge-anchoring, transcript-analysis
