# Skill Nudge Design: Exploration Order and Prescriptiveness

## Summary
When designing CLAUDE.md nudges for skill-based documentation delivery, generic "explore first, then invoke skill" instructions outperform both no-nudge and overly prescriptive instructions.

## Context
Discovered while investigating why SKILL2 (skill + CLAUDE.md nudge) approach sometimes failed to match +CLAUDE.md (embedded docs) performance in Next.js evals (specifically `029-use-cache-directive`).

## Details

### The Problem
When Claude is given documentation via a skill (requiring explicit invocation), it often:
1. Invokes the skill immediately
2. Reads docs and applies the pattern it just learned
3. Misses other requirements (like config changes) because it never explored the project

### Three Approaches Tested

| Approach | Nudge | Result |
|----------|-------|--------|
| No nudge | (none) | ❌ Claude invokes skill first → only applies code pattern → misses config |
| Prescriptive | "MUST explore next.config.ts, THEN invoke skill, THEN apply ALL requirements" | ❌ Claude follows literally → updates config → stops (tunnel vision) |
| Generic | "Before writing code, first explore the project structure, then invoke skill" | ✅ Claude thinks holistically → implements everything |

### Why Generic Works Best

1. **No tunnel vision**: Doesn't anchor Claude to specific files or steps
2. **Holistic thinking**: Claude explores naturally, builds mental model of project
3. **Discovery flow**: Similar to how +CLAUDE.md works - docs discovered during exploration

### Why Prescriptive Backfires

Overly specific instructions like "explore next.config.ts" create:
- **Cognitive anchoring**: Claude focuses on mentioned files only
- **Checklist mentality**: Claude completes literal steps, thinks task is done
- **Missing the forest**: Loses sight of the actual implementation goal

### The +CLAUDE.md Advantage

Embedding docs in CLAUDE.md/temp folders works because:
- Docs are discovered naturally during exploration
- No separate "skill invocation" step that creates cognitive separation
- Context builds organically without artificial sequencing

## Key Takeaway

**Less prescriptive = better performance** for skill nudges. Trust Claude to explore and discover rather than micro-managing the process.

```markdown
# Good nudge
Before writing code, first explore the project structure, then invoke the `skill-name` skill for documentation.

# Bad nudge (too prescriptive)
IMPORTANT: Before writing any code, you MUST:
1. First explore the project structure including specific-file.ts
2. Then invoke the `skill-name` skill to read relevant documentation
3. Apply ALL requirements from the docs, including any config changes
```

## Related
- Eval: `029-use-cache-directive`
- Tags: skills, nudges, CLAUDE.md, exploration-order, documentation-delivery
