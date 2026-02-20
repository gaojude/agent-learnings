# Next.js `instant()` E2E Testing API — Cookie-Based Protocol

**Date:** 2026-02-20
**Project:** next.js

## Overview

Next.js has an `instant()` Playwright helper for e2e tests that deterministically blocks dynamic content from streaming in, allowing assertions on loading/shell states without race conditions.

## How It Works

Uses a **cookie-based protocol** with the cookie `next-instant-navigation-testing`:

1. **Acquire lock** — sets the cookie from the page context, which triggers a `CookieStore` change event in the Next.js client runtime, acquiring an in-memory navigation lock
2. **Run callback** — dynamic data is blocked from streaming; only the static shell is visible
3. **Release lock** — clears the cookie, resolving the lock so dynamic content streams in

## Usage

```ts
await instant(page, async () => {
  await page.click('a[href="/target"]')
  // Only static shell is visible — dynamic content is blocked
  await expect(page.locator('[data-testid="loading-shell"]')).toBeVisible()
  expect(await page.locator('[data-testid="dynamic-content"]').count()).toBe(0)
})
// After exiting, dynamic content streams in
await expect(page.locator('[data-testid="dynamic-content"]')).toBeVisible()
```

## Key Details

- Works with **SPA navigations** (`<Link>`), **MPA navigations** (reload, plain `<a>`, `page.goto()`), and **initial page loads**
- Supports successive navigations within a single instant scope
- Nesting is disallowed (logs an error)
- Requires `experimental.exposeTestingApiInProductionBuild` in `next.config.js`
- Test suite: `test/e2e/app-dir/instant-navigation-testing-api/`
- The lock mechanism lives in the client runtime (`navigation-testing-lock.ts`) and `app-bootstrap.ts`
