# Debugging Docs Bias in Eval Regressions

**Date:** 2026-01-16
**Project:** next-evals-oss
**Eval:** 012-parallel-routes

## Summary

Documentation can "lock in" incorrect interpretations, preventing agents from self-correcting on retry. Both CC and +Docs made the same initial mistake, but only CC pivoted to the correct solution.

## The Prompt

From `evals/012-parallel-routes/prompt.md`:
```
Create a dashboard layout with parallel routes for \analytics\ and \settings\ tabs that can be viewed simultaneously. Use the Next.js App Router.
```

## Test Requirements

From `evals/012-parallel-routes/input/app/page.test.tsx`:
```typescript
// Lines 7-8: Check for @analytics parallel route
const analyticsPath = join(process.cwd(), 'app', '@analytics');
expect(existsSync(analyticsPath)).toBe(true);

// Lines 11-12: Check for @settings parallel route
const settingsPath = join(process.cwd(), 'app', '@settings');
expect(existsSync(settingsPath)).toBe(true);
```

The test expects parallel routes at **root level**: `app/@analytics` and `app/@settings`.

## What Each Agent Created

**CC (Claude Code without docs):**
| Attempt | Structure | Result |
|---------|-----------|--------|
| 1 | `app/dashboard/@analytics`, `app/dashboard/@settings` | FAILED |
| 2 | `app/@analytics`, `app/@settings` | PASSED |

**+Docs (Claude Code with Next.js docs):**
| Attempt | Structure | Result |
|---------|-----------|--------|
| 1 | `app/dashboard/@analytics`, `app/dashboard/@settings` | FAILED |
| 2 | `app/dashboard/@analytics`, `app/dashboard/@settings` | FAILED |
| 3 | `app/dashboard/@analytics`, `app/dashboard/@settings` | FAILED |
| 4 | `app/dashboard/@analytics`, `app/dashboard/@settings` | FAILED |
| 5 | `app/dashboard/@analytics`, `app/dashboard/@settings` | FAILED |

## Documentation Influence

The +Docs agent read `/var/folders/.../parallel-routes.mdx`. Key lines mentioning "dashboard":

```
Line 9:  "...useful for highly dynamic sections of an app, such as dashboards..."
Line 11: "For example, considering a dashboard, you can use parallel routes..."
Line 148: ```tsx filename="app/dashboard/layout.tsx" switcher
Line 163: ```jsx filename="app/dashboard/layout.js" switcher
```

The docs show `app/dashboard/layout.tsx` as an example, which combined with the prompt saying "dashboard layout" reinforced creating a `/dashboard` route segment.

## Root Cause

Both agents initially interpreted "dashboard layout" as requiring a `/dashboard` route. The difference:

- **CC** could pivot on retry because it had no strong prior belief
- **+Docs** was anchored to the documentation examples showing `app/dashboard/layout.tsx`, preventing self-correction across 5 attempts

## Key Insight

Documentation doesn't just inform - it can **anchor** the agent to specific patterns, reducing flexibility to explore alternative solutions on retry. This is especially problematic when:
1. The prompt uses terminology that appears in doc examples ("dashboard")
2. The doc examples use nested structures that don't match test expectations

## Debugging Steps Used

1. Listed output directories: `ls evals/<eval>/output-*`
2. Compared structures: `ls app/` in each output
3. Found conversation files in `~/.claude/projects/-Users-*-<eval>-output-*/`
4. Extracted docs read: `grep '"file_path":".*\.mdx"' <conversation.jsonl>`
5. Found influential lines: `grep -n "dashboard" <docs-file>`
