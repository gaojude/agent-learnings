# Async Next Browser: Cloud Architecture Vision

**Date:** 2026-03-10
**Project:** next-browser + agent-eval
**Status:** Brain dump / planning phase

## Context

Next Browser currently handles **synchronous** AI flows — developers collaboratively work with an agent on Next.js-specific problems (PPR, etc.) in real-time. The next step is exploring **asynchronous** flows.

## Core Primitive: Dev Server in Vercel Sandbox

The foundational building block is running the dev server in the cloud (Vercel sandbox).

**What's needed in the sandbox:**
1. **Dev server** — Pull from GitHub repo, `npm install`, run `next dev`
2. **Next Browser (Chromium)** — Installed alongside dev server, opens browser against it
3. **Claude Code session** — With Next Browser skills pre-loaded in context

**Reference:** agent-eval already demonstrates:
- Running Claude Code in Vercel sandbox (`@vercel/sandbox` + `claude --print`)
- Sandbox file upload and project setup
- BUT: only single-turn execution (agent gets prompt, runs to completion)

## Key Missing Piece: Multi-Turn Streaming

Agent-eval only does **one-turn** conversations. The async Next Browser needs **multi-turn conversation** with streaming updates between outside and inside the sandbox. Need to figure out how to:
- Send messages into the sandbox session
- Stream responses/updates back out
- Maintain conversation state across turns

## Two Architectural Approaches

### Option A: Full Remote (Agent Inside Sandbox)

Everything runs in the sandbox:
- Dev server + Chromium + Agent (Claude Code with skills)
- From the outside, it's just a conversation interface
- **Product vision:** Log into Vercel → pick a project → start a conversation about Next.js tasks
- Essentially building an agent product with a remote environment

**Pros:** Clean separation, agent has direct filesystem access
**Cons:** Need to solve multi-turn streaming into sandbox

### Option B: Hybrid (Agent Outside, Environment Inside)

Only browser + dev server in sandbox; agent runs locally (e.g., Claude Code on user's machine).

**Pros:** Simpler agent setup, familiar local Claude Code experience
**Cons:** Every file operation needs to be proxied as sandbox commands — reading files, editing files, running commands all need remote execution. Essentially need to replicate all of Next Browser's capabilities but over a remote boundary. Feels architecturally weird and fragile.

## Open Questions

- How to implement multi-turn streaming between outside and sandbox?
- Which architecture (A vs B) is more viable?
- How to handle Chromium setup in Vercel sandbox specifically? (agent-eval doesn't have explicit Chromium setup yet)
- How to link Vercel account + GitHub repo programmatically for sandbox setup?

## Next Steps

- [ ] Prototype dev server running in Vercel sandbox
- [ ] Explore multi-turn conversation protocol for sandbox communication
- [ ] Decide between Option A (full remote) vs Option B (hybrid)
- [ ] Investigate Chromium-in-sandbox setup (headless Chrome in Vercel sandbox)
