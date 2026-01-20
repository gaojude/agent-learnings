# Next.js Eval Test Structure: Playwright E2E + LLM Judge

## Summary
A three-layer evaluation pattern for testing AI-generated Next.js code using goal-oriented prompts, Playwright E2E tests, and LLM-as-judge natural language specs.

## Context
When evaluating whether an AI can correctly implement Next.js features, traditional regex/vitest testing is brittle and tests implementation details. This pattern tests observable behavior through a running app and allows flexible LLM evaluation of code quality.

## Details

### Directory Structure
```
evals/<eval-name>/
├── prompt.md                      # Goal-oriented prompt (WHAT, not HOW)
└── input/
    ├── app/
    │   ├── page.e2e.ts           # Playwright E2E tests
    │   └── page.spec.md          # Natural language spec for LLM judge
    └── playwright.config.ts       # Playwright configuration
```

### 1. prompt.md - Goal-Oriented Prompts

Write prompts that describe WHAT the user wants, not HOW to implement it:

```markdown
I want to build a page that supports draft mode for previewing unpublished content.

The page should:
- Display "Draft Mode: ON" in an h1 heading when draft mode is enabled
- Display "Draft Mode: OFF" in an h1 heading when draft mode is disabled
- Have an API route at /api/draft that enables draft mode and redirects to the home page
```

**Key principle**: The prompt should read like a user request, not a technical specification.

### 2. page.e2e.ts - Playwright E2E Tests

Test observable behavior through a running Next.js app:

```typescript
import { test, expect } from '@playwright/test';

test('displays draft mode OFF by default', async ({ page }) => {
  await page.goto('/');
  const heading = page.locator('h1');
  await expect(heading).toContainText('Draft Mode: OFF');
});

test('displays draft mode ON after enabling via API', async ({ page }) => {
  await page.goto('/api/draft');
  await page.waitForURL('/');
  const heading = page.locator('h1');
  await expect(heading).toContainText('Draft Mode: ON');
});
```

**Key principle**: Test what users see and experience, not implementation details.

### 3. page.spec.md - LLM Judge Criteria

Natural language requirements for LLM-based code review:

```markdown
# Evaluation Criteria

This implementation should use Next.js App Router's draft mode functionality.

## Requirements

1. Page component uses draftMode() from 'next/headers' to check if draft mode is enabled
2. Displays "Draft Mode: ON" in an h1 when draftMode().isEnabled is true
3. Displays "Draft Mode: OFF" in an h1 when draftMode().isEnabled is false
4. Has an API route at app/api/draft/route.ts that enables draft mode
5. The API route uses draftMode().enable() to enable draft mode
6. The API route redirects to '/' after enabling draft mode
7. Uses proper Next.js App Router patterns (route handlers, headers API)
```

**Key principle**: Specify correct patterns and APIs that should be used, allowing LLM to evaluate code quality beyond just "does it work".

### 4. playwright.config.ts - Standard Configuration

```typescript
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './app',
  testMatch: '**/*.e2e.ts',
  timeout: 30000,
  use: { baseURL: process.env.BASE_URL || 'http://localhost:3000' },
});
```

### Evaluation Flow

1. AI reads `prompt.md` and generates implementation
2. Build the Next.js app (`next build`)
3. Start the server (`next start`)
4. Run Playwright tests against live server
5. LLM judge reads generated code + `page.spec.md` to evaluate code quality
6. Pass = Playwright tests pass + LLM judge approves

### Benefits Over Regex/Vitest

| Aspect | Regex/Vitest | Playwright + LLM Judge |
|--------|--------------|------------------------|
| Tests | Implementation details | Observable behavior |
| Brittleness | High (regex matching) | Low (actual runtime) |
| Code quality | Not evaluated | LLM reviews patterns |
| Prompt style | Prescriptive | Goal-oriented |
| Realism | Low | High (like real users) |

## Related

- Playwright testing
- LLM-as-judge evaluation
- Next.js App Router patterns
- Goal-oriented prompt engineering
- E2E testing best practices
