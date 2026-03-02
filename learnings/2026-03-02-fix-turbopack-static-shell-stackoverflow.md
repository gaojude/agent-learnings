# Fix: Turbopack Static Shell Render Stack Overflow (Next.js)

**Date:** 2026-03-02
**Project:** next.js (vercel/next.js), ppr-optimizer
**Severity:** Breaks the Instant Navigation Testing API for any route rendered by turbopack dev server

---

## Problem Summary

When the **Instant Navigation Testing API** sends a prefetch request with the `next-instant-navigation-testing-prefetch: 1` header, the server should render a **static PPR shell** (only the prerendered portions, with Suspense fallbacks for dynamic content). Instead, the server crashes with:

```
RangeError: Maximum call stack size exceeded
```

This causes a **500 response** for every testing prefetch request. The client-side testing lock mechanism then falls through to a code path that fetches full data — making the PPR shell screenshot show the fully-resolved page instead of skeletons.

---

## Background: How Client-Nav PPR Shell Capture Works

### The Two Shell Modes (do NOT confuse them)

1. **Initial mode** (`page.goto` with cookie): Server sees cookie in HTTP request → responds with raw PPR shell HTML containing `<template>` holes. Hydration is skipped. Metrics: template hole count, body text length.

2. **Client-nav mode** (`router.push` with cookie lock): Page is already hydrated from a normal entry page load. Cookie activates the navigation-testing-lock. `router.push()` triggers client-side navigation. The lock blocks `writeDynamicDataIntoNavigationTask()` so React renders **Suspense fallbacks** instead of resolved data. Metrics: Suspense boundary count, suspended count, visual screenshot.

### The Navigation Testing Lock Mechanism

**File:** `next.js/.../segment-cache/navigation-testing-lock.ts`

- `startListeningForInstantNavigationCookie()` registers a `cookieStore` change listener
- When cookie `next-instant-navigation-testing` is set → `acquireLock()` creates a Promise
- `isNavigationLocked()` checks `document.cookie` synchronously
- `waitForNavigationLockIfActive()` awaits `lockState.promise` (blocks until cookie removed)

### The Navigation Flow (segment cache path)

**File:** `next.js/.../segment-cache/navigation.ts`

```
router.push(url)
  → navigate()
    → cache hit? → navigateUsingPrefetchedRouteTree() [seed data: null]
    → cache miss? → navigateToUnknownRoute()
        → if testing lock active:
            → tryNavigateUsingTestingAPIPrefetch()
                → schedulePrefetchTask() with testing header
                → server renders STATIC SHELL (isDebugStaticShell=true)
                → cache populated with static-only data
                → navigateUsingPrefetchedRouteTree() [seed data: null]
            → if prefetch fails: returns null, falls through ↓
        → fetchServerResponse() [FULL data, no testing header]
        → convertServerPatchToFullTree() → seed WITH data
  → navigateToKnownRoute(seed)
    → startPPRNavigation(seedData, seedHead)
      → createCacheNodeForSegment(seedRsc)
        → if seedRsc !== null:
            rsc = seedRsc           // ← data rendered IMMEDIATELY
            needsDynamicRequest = false  // ← lock NEVER checked
        → if seedRsc === null:
            rsc = createDeferredRsc()  // ← pending promise
            needsDynamicRequest = true // ← lock WILL block this
    → spawnDynamicRequests()
      → fetchMissingDynamicData()
        → fetchServerResponse() [dynamic segments only]
        → await waitForNavigationLock()  // ← LOCK CHECK HERE
        → writeDynamicDataIntoNavigationTask()
```

**Key insight:** The lock only blocks `writeDynamicDataIntoNavigationTask()`. When the testing prefetch FAILS, the fallback path puts full data into `seedRsc`, which is used directly — the lock is never consulted.

### Server-Side: How the Testing Header Triggers Static Shell Mode

**File:** `next.js/.../build/templates/app-page.ts` (lines 340-378)

```typescript
const couldSupportPPR = checkIsAppPPREnabled(nextConfig.experimental.ppr)
// ↑ With cacheComponents: true, config normalization sets experimental.ppr = true

const isInstantNavigationTest =
    exposeTestingApi &&       // true in dev mode
    couldSupportPPR &&        // true (cacheComponents → experimental.ppr = true)
    req.headers['next-instant-navigation-testing-prefetch'] === '1'

const isRoutePPREnabled =
    couldSupportPPR &&
    (prerenderManifest...PARTIALLY_STATIC ||
      (isInstantNavigationTest && exposeTestingApi))  // ← dev fallback

const isDebugStaticShell =
    isInstantNavigationTest && isRoutePPREnabled  // → TRUE
```

When `isDebugStaticShell = true`, the render options become:
```typescript
{
    isBuildTimePrerendering: true,
    supportsDynamicResponse: false,
    isStaticGeneration: true,
}
```

This forces the server to render as if it's a static build — any call to `cookies()`, `headers()`, etc. causes the render to "postpone" at the Suspense boundary (rendering the fallback instead).

---

## The Bug: Stack Overflow in Static Shell Render

When the server renders with `isDebugStaticShell = true` for the deployments route, turbopack's module loading enters infinite recursion:

```
RangeError: Maximum call stack size exceeded
  at [turbopack]_runtime.js:1215
  at instantiateModuleShared → instantiateModule
  → getOrInstantiateModuleFromParent → commonJsRequire
  → requireModule → initializeModuleChunk → readChunk
  → getComponentNameFromType → retryNode → renderNodeDestruct...
```

The cycle: `retryNode` → `getComponentNameFromType` → `readChunk` → `initializeModuleChunk` → `requireModule` → `commonJsRequire` → `getOrInstantiateModuleFromParent` → `instantiateModuleShared` → (back to start)

This is a **circular module dependency** that only manifests when turbopack renders in static generation mode. Normal (dynamic) renders don't trigger it because the module loading takes a different path.

### Verified with direct HTTP request:

```
Regular Prefetch (no testing header): 200 OK, 251,731 bytes (RSC flight data)
Testing Prefetch (with testing header): 500 Error, 20,201 bytes (HTML error page)
Full Dynamic (no prefetch header):     200 OK, 319,510 bytes (RSC flight data)
```

---

## Consequence Chain

1. `tryNavigateUsingTestingAPIPrefetch()` schedules 17 segment prefetch requests with testing header
2. **ALL return 500** (static shell render crashes)
3. `tryNavigateUsingTestingAPIPrefetch()` → cache empty → returns `null`
4. Falls through to `fetchServerResponse()` without testing header → gets **319KB of full data**
5. `convertServerPatchToFullTree()` creates seed with full data
6. `startPPRNavigation(seedData)` → `createCacheNodeForSegment(seedRsc)` populates CacheNode tree immediately
7. `seedRsc !== null` → `doesSegmentNeedDynamicRequest = false` → **lock never checked**
8. 65+ boundaries render with full deployment data
9. Only 4 segments where `seedRsc === null` → deferred promises → lock blocks → stay suspended
10. **PPR shell screenshot shows full page instead of skeleton**

### Why it appeared to work before commit `f1122481f7b`:

That commit changed the deployments layout from `withInterceptors` (which called `cookies()`, making it async/dynamic) to a synchronous layout. Before: the outer `<Suspense fallback={<DeploymentsSkeleton />}>` always suspended during client nav because the auth was async — masking the data leak. After: layout renders synchronously → data leak becomes visible.

The static shell crash was **already happening** before that commit. It was just hidden by the outer Suspense boundary.

---

## Plan to Fix (in Next.js)

### Step 1: Reproduce the crash in isolation

Create a minimal reproduction:
- A Next.js app with `cacheComponents: true` using turbopack dev server
- A route with enough component imports to trigger circular module resolution
- Send a request with `next-instant-navigation-testing-prefetch: 1` header
- Confirm 500 response with stack overflow

### Step 2: Locate the turbopack module resolution cycle

The stack trace shows the cycle starts at `getComponentNameFromType` in the React server renderer. During static shell render (`isStaticGeneration: true`), React tries to get component names (likely for error messages or dev warnings). This triggers `readChunk` → `initializeModuleChunk` → `requireModule` → turbopack's `commonJsRequire` → `getOrInstantiateModuleFromParent` → `instantiateModuleShared`.

**Key files to investigate:**
- `[turbopack]_runtime.js` — `instantiateModuleShared` (line 1213), `getOrInstantiateModuleFromParent` (line 1471)
- React server renderer — `getComponentNameFromType`, `retryNode`, `renderNodeDestructive`
- `app-page-turbo-experimental.runtime.dev.js` — `requireModule` (line 75:42664)

The circular dependency is likely between modules that import each other. In normal (dynamic) render mode, the modules are already instantiated by the time they're needed. In static generation mode, the render path triggers module loading in a different order, exposing the cycle.

### Step 3: Identify the specific modules involved

Add a breadcrumb/guard in turbopack's `instantiateModuleShared` to detect re-entrant module loading:

```javascript
// In [turbopack]_runtime.js, around line 1213
if (module.__instantiating) {
    console.error('Circular module instantiation detected:', moduleId);
    return module; // Break the cycle
}
module.__instantiating = true;
// ... existing code ...
delete module.__instantiating;
```

This will log which module(s) form the cycle.

### Step 4: Fix the cycle

Once the specific modules are identified, the fix depends on the nature of the cycle:

**Option A: Fix in turbopack runtime** — Add cycle detection/breaking in `getOrInstantiateModuleFromParent`. When a module is already being instantiated, return a partial module (with `exports = {}`) to break the cycle, similar to how Node.js handles circular `require()`.

**Option B: Fix in React's `getComponentNameFromType`** — This function is called during rendering to get display names. If it triggers module loading (via `readChunk`), it should handle the case where the module can't be loaded (return a fallback name instead of recursing).

**Option C: Fix in the route module structure** — If specific routes have circular imports, restructure them. But this is unlikely to be route-specific given it's a turbopack runtime issue.

### Step 5: Verify the fix

After fixing:
1. Send testing prefetch request → should get 200 with small RSC response (static shell only)
2. Run client-nav PPR shell capture → should show skeletons, not full data
3. Compare response sizes: testing prefetch should be significantly smaller than regular prefetch
4. Run the ppr-optimizer ralph loop → should correctly detect shell changes

---

## Config Context

**Front repo (`next.config.mjs`):**
```javascript
cacheComponents: true,  // ← enables PPR (merged from experimental.ppr)
experimental: {
    // NO explicit ppr: true (it's deprecated, use cacheComponents)
    exposeTestingApiInProductionBuild: process.env.NEXT_PUBLIC_TESTMODE === 'true',
}
```

**Config normalization** (`next.js/.../server/config.ts:1385-1387`):
```typescript
if (result.cacheComponents) {
    result.experimental.ppr = true  // ← cacheComponents implies PPR
}
```

This means `checkIsAppPPREnabled(nextConfig.experimental.ppr)` returns `true` at runtime, so `couldSupportPPR = true` and the testing API IS enabled. The issue is purely in the static shell render crashing, not in config.
