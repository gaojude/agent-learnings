# CLAUDE.md Instructions to Boost Skill Trigger Rate

## Summary
Use directive language with "always", "first", and motivation about outdated training data to reliably trigger Claude to use skills.

## Context
When creating a SKILL.md for Claude Code, the skill may not be triggered automatically. Adding a CLAUDE.md file with the right wording can significantly boost the skill trigger rate.

## Details

### What works
```markdown
# Next.js Project

Before starting any Next.js task, always use the `nextjs-doc` skill first. Your training data may be outdated.
```

Key elements:
- **"always"** - makes it a consistent rule
- **"first"** - establishes priority/sequence
- **"Your training data may be outdated"** - provides motivation for why the skill matters

### What doesn't work

**Too minimal:**
```markdown
For any Next.js task, use the `nextjs-doc` skill to ensure grounded knowledge.
```
This is too passive and doesn't trigger the skill.

**Too verbose:**
Duplicating all the SKILL.md instructions (what command to run, what the output looks like, etc.) is unnecessary - the agent reads SKILL.md for that detail.

### Test results
- `+SKILL.md` alone: ⚠️ (skill not used in 56% of cases)
- `+SKILL2` (SKILL.md + boosted CLAUDE.md): 📚 (docs pulled & read)

## Related
- Creating Claude Code skills
- SKILL.md structure
- Next.js evals comparison
