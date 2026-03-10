# Async Next Browser: Running an AI Dev Environment in the Cloud

**Date:** 2026-03-10
**Project:** next-browser + agent-eval
**Status:** Brain dump / planning phase

## Where We Are Today

We've nailed down the synchronous part of Next Browser. Today, Next Browser is an interactive AI flow that lets developers collaboratively work with an agent to solve Next.js-specific problems — things like PPR (Partial Prerendering), server components, caching, and other framework-level challenges that are hard to get right. The developer and the agent work together in real-time: the agent looks at your code, runs the dev server, opens a browser, inspects the result, and helps you iterate.

This works well. But it's inherently synchronous — you're sitting there, in a conversation, waiting for each step. The next thing to explore is: what does an **asynchronous** version of this look like?

## The First Primitive: Dev Server in the Cloud

The foundational building block for any async flow is the ability to run the dev server remotely — specifically, in a Vercel sandbox. If we can get the dev server running in the cloud, we can build everything else on top of that.

We already have a reference for how to do parts of this. The agent-eval project demonstrates running Claude Code inside a Vercel sandbox using `@vercel/sandbox`. It shows how to upload project files, install dependencies, and execute agent commands in an isolated environment. So we know the sandbox infrastructure works.

Now imagine extending that. You link your Vercel account, pull from your GitHub repo, run `npm install`, and start `next dev` — all inside the sandbox. You also install Next Browser alongside it, which gives you Chromium running against the dev server (there's a specific way to set up headless Chromium in the sandbox; agent-eval has patterns for this). And finally, you spin up a Claude Code session inside the sandbox, pre-loaded with all the Next Browser skills in its context window.

At that point, you have a complete, self-contained AI development environment running in the cloud: a dev server, a browser that can inspect it, and an agent that knows how to use both.

## The Missing Piece: Multi-Turn Conversations

Here's where things get interesting — and where agent-eval falls short as a reference. Agent-eval only does **one-turn** conversations. You send the agent a prompt, it runs to completion, and you get the result. That's fine for evaluations, but it's not what we want here.

For an async Next Browser, we need **multi-turn conversations** that stream back and forth between the outside world and the sandbox. A developer should be able to start a conversation, get a response, ask follow-up questions, provide corrections, and watch the agent iterate — all while the agent is operating inside this remote environment. We need to figure out the protocol for passing messages in and streaming updates back out, while maintaining conversation state across turns.

## Two Ways to Architect This

There are two fundamentally different ways to set this up, and they have very different tradeoffs.

### Option A: Everything in the Sandbox

In this approach, the agent lives inside the sandbox alongside the dev server and browser. From the outside, you're just talking to an API — you send messages in, you get responses back. The sandbox is a fully self-contained environment.

This leads to a pretty clean product vision: you log into Vercel, choose a project, and start a conversation. The conversation is about Next.js tasks — "help me set up PPR," "why is this server component re-rendering," "optimize this route" — and behind the scenes, the agent is working in a real environment with your actual code, a running dev server, and a browser to verify its work.

You're essentially building an agent product. From the user's perspective, it's just a chat interface. All the complexity — the dev server, the browser, the file system, the skills — is hidden behind the sandbox boundary.

### Option B: Agent Outside, Environment Inside

The alternative is to keep the agent on the user's side — running locally in Claude Code, for instance — but have the dev server and browser running remotely in the sandbox. The agent does the same things Next Browser does today, but instead of operating on local files and a local dev server, it manipulates things inside the sandbox.

This sounds simpler at first, but it gets weird quickly. Today, when the agent needs to read a file, it just reads it. When it needs to edit code, it just edits it. When it needs to check the browser, it opens Chromium locally. In this hybrid model, every single one of those operations needs to be proxied through the sandbox boundary. Reading a file becomes "send a read command to the sandbox." Editing a file becomes "send an edit command to the sandbox." You'd essentially need to replicate the entire Next Browser command surface, but over a remote connection.

That's not just extra engineering work — it's architecturally fragile. You're adding a network boundary in the middle of what should be tight, low-latency interactions between the agent and its environment. It feels like the wrong abstraction.

## What's Next

The cleaner path seems to be Option A — put the agent in the sandbox with everything else, and figure out the multi-turn streaming problem. The key things to figure out are:

1. Getting the dev server reliably running in the Vercel sandbox (pulling from GitHub, installing deps, starting `next dev`)
2. Setting up Chromium inside the sandbox so Next Browser can do its thing
3. Building a multi-turn conversation protocol that lets us stream messages in and out of the sandbox
4. Pre-loading the Claude Code session with Next Browser skills so the agent has full capabilities from the start

None of these are solved yet, but none of them are mysteries either — they're engineering problems with clear shapes. The agent-eval codebase gives us a starting point for most of them; the main gap is the multi-turn piece.
