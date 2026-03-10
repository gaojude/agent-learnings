# Async Next Browser: Cloud Architecture Vision

**Date:** 2026-03-10
**Project:** next-browser + agent-eval
**Status:** Prototyped and validated. All 4 prototypes passing.

## Where We Are Today

We've nailed down the synchronous part of Next Browser. Today, Next Browser is an interactive AI flow that lets developers collaboratively work with an agent to solve Next.js-specific problems — things like PPR (Partial Prerendering), server components, caching, and other framework-level challenges that are hard to get right. The developer and the agent work together in real-time: the agent looks at your code, runs the dev server, opens a browser, inspects the result, and helps you iterate.

This works well. But it's inherently synchronous — you're sitting there, in a conversation, waiting for each step. The next thing to explore is: what does an **asynchronous** version of this look like?

## The First Primitive: Dev Server in the Cloud

The foundational building block for any async flow is the ability to run the dev server remotely — specifically, in a Vercel sandbox. If we can get the dev server running in the cloud, we can build everything else on top of that.

**Validated:** Prototype 1 demonstrates this works in 18 seconds from zero to a running dev server with a public URL (`https://sb-*.vercel.run`).

## Chrome in the Sandbox

Headless Chromium runs in the Vercel sandbox using `@sparticuz/chromium` (pre-compiled for serverless). System dependencies are installed via `sudo dnf install` on Amazon Linux 2023. puppeteer-core connects via CDP WebSocket.

**Validated:** Prototype 2 takes a pixel-perfect screenshot of the Next.js dev server from headless Chrome running inside the sandbox. 44 seconds total (24s of that is Chrome install, eliminable with snapshots).

## Multi-Turn Conversations: Solved

The brain dump identified multi-turn streaming as the key missing piece. Turns out, the Anthropic SDK's standard messages API with tool use handles this natively. No custom WebSocket/SSE protocol needed.

**Architecture:** Agent runs outside the sandbox. Sandbox is exposed as tools (run_shell, read_file, write_file, browser_navigate, browser_screenshot, browser_get_text, browser_console_errors). Each turn: user message -> agent response with tool calls -> execute tools in sandbox -> return results -> continue.

**Validated:** Prototypes 3 and 4 demonstrate multi-turn conversations where the agent explores files, creates pages, navigates the browser, takes screenshots, and verifies its own work.

## Architecture Decision: Modified Option A

Two architectures were considered:
- **Option A (Full Remote):** Everything in sandbox, agent drives from outside via tools
- **Option B (Hybrid):** Agent runs locally, proxies every operation into sandbox

**Winner: Modified Option A.** The environment (dev server, Chrome) runs in the sandbox. The agent runs outside but operates on the sandbox via Anthropic SDK tool calls. This avoids the "every file operation is a proxy" problem from Option B — the tools are trivially simple, and the SDK handles conversation state.

## Performance

| Operation                | Time  |
|--------------------------|-------|
| Sandbox creation         | ~2s   |
| create-next-app          | ~12s  |
| Chrome install           | ~24s  |
| Chrome launch + CDP      | ~2.5s |
| Dev server startup       | <1s   |
| Full cold start          | ~40s  |
| With snapshots (est.)    | ~5s   |

## What's Next

1. **Sandbox snapshots** — Pre-bake Chrome + system deps, get cold start from 40s to ~5s
2. **Next Browser in sandbox** — Install the actual Next Browser daemon inside for DevTools + PPR
3. **Git source** — Clone user's repo instead of scaffolding
4. **Streaming UI** — Web interface that streams the conversation
5. **React DevTools via CDP** — Investigate getting DevTools data through CDP without extensions

## Prototypes

Located at `next-browser/prototypes/cloud/`. Run with `npm run proto:sandbox|chrome|agent|full`.
Full technical report at `prototypes/cloud/REPORT.md`.
