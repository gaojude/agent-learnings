# Next.js Eval Test Structure with Playwright E2E and LLM Judge

## Summary
A three-layer evaluation system for testing LLM code generation: goal-oriented prompts, Playwright E2E tests for behavior, and LLM judge for code quality.

## Context
When building evals for LLM code generation, you need to test both that the code **works** (behavior) and that it follows **best practices** (patterns). This structure separates those concerns.

## Details

### Directory Structure

```
evals/
  XXX-eval-name/
    prompt.md              # Goal-oriented task description (visible to agent)
    input/
      app/
        page.tsx           # Starting code scaffold
        page.e2e.ts        # Playwright E2E tests (hidden from agent)
        page.spec.md       # LLM judge criteria (hidden from agent)
        layout.tsx
      package.json
      playwright.config.ts # Playwright config (hidden from agent)
```

### Three Key Files

#### 1. `prompt.md` - Goal-Oriented Task
Describe **what** the user wants, not **how** to implement:

```markdown
# Good
Create a click counter page. Display "Count: 0" initially with an
"Increment" button that increases the count.

# Bad (too prescriptive)
Create a client component with 'use client' directive. Use useState...
```

#### 2. `page.e2e.ts` - Behavior Tests
Test observable behavior with Playwright:

```typescript
import { test, expect } from '@playwright/test';

test('counter increments on click', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('text=Count: 0')).toBeVisible();
  await page.click('button');
  await expect(page.locator('text=Count: 1')).toBeVisible();
});
```

#### 3. `page.spec.md` - Judge Criteria
Natural language criteria for LLM judge:

```markdown
# Evaluation Criteria

## Requirements
1. Uses 'use client' directive - required for client-side interactivity
2. Uses React useState hook to manage count state, initialized to 0
3. Displays current count in format "Count: X"
4. Has button labeled "Increment" that increases count by 1
5. Component is the default export
```

### Evaluation Flow

```
Agent generates code (sees prompt.md + page.tsx)
    ↓
Build (next build)
    ↓
Lint (eslint)
    ↓
E2E Tests (playwright against next start)
    ↓
LLM Judge (evaluates code against page.spec.md)
    ↓
Score: Build ✓ Lint ✓ Test ✓ Judge: 85%
```

### File Visibility

| File | Visible to Agent | Purpose |
|------|-----------------|---------|
| `prompt.md` | ✅ Yes | Task description |
| `page.tsx` | ✅ Yes | Starting code |
| `page.e2e.ts` | ❌ No | Behavior testing |
| `page.spec.md` | ❌ No | Judge evaluation |
| `playwright.config.ts` | ❌ No | Test config |

### Playwright Config

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './app',
  testMatch: '**/*.e2e.ts',
  timeout: 30000,
  use: { baseURL: process.env.BASE_URL || 'http://localhost:3000' },
});
```

### LLM Judge Implementation

The judge uses Claude Sonnet to evaluate code against natural language criteria:

```typescript
const JUDGE_SYSTEM_PROMPT = `You are an expert code reviewer...
Respond in JSON: { "score": 0.85, "criteria_met": [...], "criteria_failed": [...] }`;

const judgePrompt = `
## Specification
${specContent}

## Implementation
${projectFiles}

Evaluate how well this implementation meets the criteria.`;
```

### Why This Structure Works

1. **Prompts test real-world usage** - Users describe goals, not implementation
2. **E2E tests verify behavior** - Does it actually work?
3. **Judge verifies patterns** - Does it follow Next.js best practices?
4. **Separation of concerns** - Behavior vs code quality are independent

## Related
- Playwright testing
- LLM-as-judge evaluation
- Next.js App Router patterns
- Goal-oriented prompting
