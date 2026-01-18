# Skill Exploration Order: The "Knowledge Anchor" Effect

## Summary

When Claude reads documentation/skill content BEFORE exploring the codebase, it anchors to the documentation's approach and fails to adopt codebase-specific patterns - even after multiple retry attempts.

## Context

Discovered while investigating why the `021-avoid-fetch-in-effect` eval consistently failed with SKILL2 configuration (skill + CLAUDE.md nudge to use skill) while passing with other configurations.

**Date:** 2025-01-17

**Eval:** `021-avoid-fetch-in-effect` in `next-evals-oss`

## The Problem

The eval requires creating a `UserProfile` component that fetches from `/api/users/profile`. The existing codebase (`page.tsx`) has a critical pattern:

```typescript
// From evals/021-avoid-fetch-in-effect/input/app/page.tsx
async function getProducts() {
  try {
    const res = await fetch('https://api.example.com/products');
    return res.json();
  } catch {
    // Return mock data for build time
    return [{ id: 1, name: 'Sample Product' }];
  }
}
```

The **try-catch with fallback** is required because during `next build`, the API endpoint doesn't exist, causing the fetch to fail. Without error handling, the build fails.

## Configurations Tested

| Config | CLAUDE.md Instruction | Result |
|--------|----------------------|--------|
| Baseline | None | Pass (after retries) |
| +CLAUDE.md | Docs bundled in CLAUDE.md | Pass (after retries) |
| SKILL.md | Skill available but NOT invoked | Pass (1st attempt) |
| **SKILL2** | "Before starting any Next.js task, use the skill" | **FAIL (all 5 attempts)** |
| **SKILL3** | "MANDATORY: invoke skill FIRST, do NOT explore codebase" | **FAIL (all 5 attempts)** |
| **SKILL4** | "FIRST explore codebase, THEN use skill" | **PASS (1st attempt)** |

## Hypothesis Testing

### H1: Knowledge Anchor Effect
**Hypothesis:** When Claude reads docs first, it anchors to the "canonical" documentation approach and deprioritizes codebase patterns.

**Test:** SKILL3 - Force skill invocation before any codebase exploration.

**CLAUDE.md used:**
```markdown
# Next.js Project

MANDATORY: You MUST invoke the `nextjs-doc` skill IMMEDIATELY as your FIRST action for ANY Next.js related task.
Do NOT explore the codebase until you have consulted the official docs via the skill.
Your training data is OUTDATED - the skill has the current correct patterns.
```

**Result:** FAILED all 5 attempts. Never added try-catch.

### H2: Docs-First Ordering
**Hypothesis:** The order of information consumption determines which pattern Claude adopts.

**Test:** SKILL4 - Instruct to explore codebase FIRST, then use skill.

**CLAUDE.md used:**
```markdown
# Next.js Project

IMPORTANT: When working on Next.js tasks, ALWAYS follow this order:
1. FIRST: Explore the existing codebase to understand current patterns and conventions
2. THEN: Use the `nextjs-doc` skill if you need to clarify Next.js APIs or features

The existing codebase patterns should be your PRIMARY guide. Use the skill docs only for reference.
Code examples in docs are just examples - don't confuse with project code. Match what the project already does.
```

**Result:** PASSED on 1st attempt. Correctly added try-catch.

## Actual Generated Code

### SKILL3 (docs-first) - Attempt 1 of 5
**File:** `evals/021-avoid-fetch-in-effect/output-claude-code-nextjs-skill3-1768695814764/app/UserProfile.tsx`
```typescript
interface User {
  name: string;
  email: string;
}

async function getUserProfile(): Promise<User> {
  const res = await fetch('/api/users/profile');
  return res.json();
}

export default async function UserProfile() {
  const user = await getUserProfile();

  return (
    <div>
      <h2>User Profile</h2>
      <p>Name: {user.name}</p>
      <p>Email: {user.email}</p>
    </div>
  );
}
```
**NO TRY-CATCH** - Build fails during `next build`

### SKILL3 (docs-first) - Attempt 5 of 5
**File:** `evals/021-avoid-fetch-in-effect/output-claude-code-nextjs-skill3-1768695958358-attempt5/app/UserProfile.tsx`
```typescript
interface User {
  name: string;
  email: string;
}

async function getUserProfile(): Promise<User> {
  const res = await fetch('/api/users/profile');
  return res.json();
}

export default async function UserProfile() {
  const user = await getUserProfile();

  return (
    <div>
      <h2>User Profile</h2>
      <p>Name: {user.name}</p>
      <p>Email: {user.email}</p>
    </div>
  );
}
```
**IDENTICAL to attempt 1** - Never learned from failures

### SKILL4 (codebase-first) - Attempt 1 of 1
**File:** `evals/021-avoid-fetch-in-effect/output-claude-code-nextjs-skill4-1768695815854/app/UserProfile.tsx`
```typescript
interface User {
  name: string;
  email: string;
}

async function getUserProfile(): Promise<User> {
  try {
    const res = await fetch('/api/users/profile');
    return res.json();
  } catch {
    return { name: 'Guest', email: '' };
  }
}

export default async function UserProfile() {
  const user = await getUserProfile();

  return (
    <div>
      <h2>User Profile</h2>
      <p>Name: {user.name}</p>
      <p>Email: {user.email}</p>
    </div>
  );
}
```
**HAS TRY-CATCH WITH FALLBACK** - Build succeeds

## Transcript Evidence

From SKILL2 conversation transcript (docs-first):
> "I'll start by using the `nextjs-doc` skill to check the current Next.js documentation... then explore the codebase"

From Baseline conversation transcript (codebase-first):
> "Includes error handling with fallback values, matching the pattern in `page.tsx`"

## Root Cause Analysis

When Claude reads documentation **before** exploring the codebase:

1. **Anchoring Bias**: Claude anchors to the documentation's "canonical" approach (async Server Components with simple fetch)
2. **Authority Weighting**: Documentation is perceived as authoritative, so its patterns are prioritized
3. **Pattern Blindness**: Even when Claude later sees the try-catch in `page.tsx`, it doesn't adopt it because it has already committed to the docs approach
4. **Retry Ineffectiveness**: Each retry starts fresh but with the same CLAUDE.md instruction, so Claude repeats the same docs-first exploration and produces identical code

## Recommendation

For skills that provide reference documentation, the CLAUDE.md instructions should explicitly prioritize codebase exploration:

**BAD (causes anchor effect):**
```markdown
Before starting any Next.js task, use the `nextjs-doc` skill for official Next.js reference.
```

**GOOD (prevents anchor effect):**
```markdown
IMPORTANT: When working on Next.js tasks, ALWAYS follow this order:
1. FIRST: Explore the existing codebase to understand current patterns and conventions
2. THEN: Use the `nextjs-doc` skill if you need to clarify Next.js APIs or features

The existing codebase patterns should be your PRIMARY guide. Use the skill docs only for reference.
```

## Test Commands Used

```bash
# H1 test - force docs first (FAILED)
bun cli.ts --eval 021-avoid-fetch-in-effect --claude-code --skill3 --debug --verbose

# H2 test - codebase first (PASSED)
bun cli.ts --eval 021-avoid-fetch-in-effect --claude-code --skill4 --debug --verbose
```

## Related

- Claude Code skills system
- Next.js server components data fetching
- Cognitive anchoring bias in LLMs
- Eval design for testing agent behavior

## Tags

`#skills` `#claude-code` `#next-js` `#eval` `#anchoring-bias` `#exploration-order`
