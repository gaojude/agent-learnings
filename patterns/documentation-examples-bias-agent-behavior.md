# Documentation Examples Can Bias Agent Behavior

## Summary
Specific code examples in documentation (especially file paths) can anchor AI agents toward replicating exact structures, even when inappropriate for the actual task.

## Context
Discovered while investigating why a Next.js parallel routes eval regressed when official docs were added to Claude Code's context. The eval expected root-level parallel routes (`app/@analytics/`) but Claude consistently created nested routes (`app/dashboard/@analytics/`).

## Details

### The Problem
When docs showed examples like:
```
app/dashboard/layout.tsx
app/dashboard/@analytics/
app/dashboard/@settings/
```

Claude followed this **nested pattern** even though:
1. The word "dashboard" came from the prompt, not docs
2. The test expected root-level routes
3. Root-level examples also existed in the docs

### Key Findings

#### 1. Structural Patterns Anchor Behavior
- Agents follow the **structure** shown in examples, not just copy words
- Renaming "dashboard" to "admin-panel" in docs had zero effect
- The nested `app/folder/@slot` pattern was the issue, not the folder name

#### 2. Docs Suppress Exploration
| Scenario | Behavior | Result |
|----------|----------|--------|
| Without docs | Agent explores codebase, sometimes finds test file | ~67% success |
| With docs | Agent feels confident, skips exploration | ~20% success |

The "false confidence" effect: docs make agents think they know the answer, so they skip reading existing code/tests that would reveal the expected structure.

#### 3. Generic Examples Perform Better
When docs were modified to show only root-level examples:
```
app/layout.tsx  (instead of app/dashboard/layout.tsx)
```
The success rate improved - the +Docs run eventually passed.

### Testing Methodology
```typescript
// docsModifier callback pattern for testing doc modifications
async function makeDocsRootLevel(claudeMdPath: string): Promise<void> {
  // Find and modify the docs file
  content = content.replace(
    /```tsx filename="app\/[a-z-]+\/layout\.tsx" switcher/g,
    '```tsx filename="app/layout.tsx" switcher'
  );
}

// Use in eval runner
runClaudeCodeEval(evalPath, {
  nextjsDocs: true,
  docsModifier: makeDocsRootLevel
});
```

### Recommendations for Documentation Authors

1. **Use minimal/generic paths** - Prefer `app/layout.tsx` over `app/my-feature/layout.tsx`
2. **Show the simplest case first** - Root-level before nested examples
3. **Avoid domain-specific folder names** - `folder/` instead of `dashboard/` or `admin/`
4. **Include "when to use" guidance** - Help agents understand which pattern fits their situation

### Recommendations for Agent/Tool Developers

1. **Encourage exploration** - Add prompts like "Check existing code structure before implementing"
2. **Don't over-trust docs** - Docs provide patterns, but codebase is source of truth
3. **Test with/without docs** - Compare behavior to identify docs-induced bias

## Related
- Next.js App Router parallel routes
- Claude Code CLAUDE.md documentation injection
- Eval testing methodology
- Agent behavior analysis
