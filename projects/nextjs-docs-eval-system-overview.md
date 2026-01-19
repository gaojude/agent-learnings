# Next.js Documentation Eval System: Testing How to Give AI Coding Agents Reference Docs

## Summary

A test harness measuring whether giving Claude Code access to Next.js documentation improves coding task completion, and comparing different methods of providing that documentation.

## Context

When building AI coding agents, a key question is: should we give them access to documentation? And if so, how? This eval system tests 4 different approaches across 50 Next.js coding tasks.

## The Core Question

> When an AI coding agent needs to implement something in Next.js, does giving it access to official documentation help? And if so, **how** should the docs be provided?

## The Eval Setup

**50 Next.js coding tasks** ranging from simple to complex:
- Simple: "Create a server component" (001)
- Medium: "Implement parallel routes for a dashboard" (012, 039)
- Hard: "Use the new 'use cache' directive correctly" (029)
- AI SDK: "Migrate from OpenAI SDK to Vercel AI SDK" (031-038)

**Each task has:**
- `input/` - starter code (a Next.js project)
- `prompt.md` - the task description
- `input/app/*.test.tsx` - Vitest tests that verify correctness

**Success criteria:**
- ✅ Build passes (`next build`)
- ✅ Lint passes (`eslint`)
- ✅ Tests pass (`vitest`)

## The Four Approaches Being Compared

### 1. Baseline (No Docs)
Just run Claude Code on the task with no documentation provided.
```
Claude Code + Task Prompt → Output
```

### 2. +CLAUDE.md (Docs Index in Context)
A `CLAUDE.md` file is added to the project root containing an **index of all Next.js doc file paths**. The agent can `Read` any doc file directly.
```
CLAUDE.md contains:
  # Next.js Project
  Docs available below for reference.

  ## Documentation Index
  - [Parallel Routes](/path/to/parallel-routes.mdx)
  - [Use Cache](/path/to/use-cache.mdx)
  - ... (~200 doc file paths)
```

### 3. +SKILL.md (On-Demand Skill)
A Claude Code "skill" is installed at `.claude/skills/nextjs-doc/SKILL.md`. The agent must **invoke the skill** to access docs.
```
.claude/skills/nextjs-doc/SKILL.md contains:
  ---
  name: nextjs-doc
  description: "Next.js official documentation for reference."
  ---

  ## Documentation Index
  - [Parallel Routes](/path/to/parallel-routes.mdx)
  ...
```
The agent has to call the Skill tool to load this.

### 4. +SKILL2 (Skill + CLAUDE.md Mention)
Same as SKILL.md, but the project also has a `CLAUDE.md` that mentions the skill exists:
```
CLAUDE.md contains:
  # Next.js Project
  The `nextjs-doc` skill is available for official documentation reference.
```

## How Docs Are Generated

Uses `npx @judegao/next-skills` which:
1. Detects the project's Next.js version from package.json
2. Downloads matching official docs from Vercel
3. Creates either a CLAUDE.md (index) or SKILL.md (skill-based) setup

## Running the Evals

```bash
# Run all 50 evals comparing all 4 approaches
bun cli.ts --claude-code --all --compare-nextjs-docs

# Run specific eval
bun cli.ts --claude-code --eval 029-use-cache-directive --compare-nextjs-docs

# Keep output folders for debugging
bun cli.ts --claude-code --eval 012-parallel-routes --compare-nextjs-docs --debug
```

## Reading the Results

```
| Eval                    | Claude Code | +CLAUDE.md | +SKILL.md | +SKILL2  |
|-------------------------|-------------|------------|-----------|----------|
| 029-use-cache-directive | ❌✅✅ 🔄❌    | ✅✅✅        | ✅✅✅ 🔄✅ ⚠️ | ✅✅✅ 📚   |
```

**Emoji meanings:**
- `✅✅✅` = Build/Lint/Test all pass
- `❌✅✅` = Build failed, Lint passed, Test passed
- `🔄✅` = Failed initially but passed on retry
- `🔄❌` = Failed even after retries
- `📚` = Skill invoked AND docs were read
- `📥` = Skill invoked but docs not read
- `⚠️` = Skill was available but never invoked

## Key Findings So Far

**Results after multiple runs:**
| Approach | Pass Rate |
|----------|-----------|
| Baseline | 32-33/50 |
| +CLAUDE.md | 41-42/50 |
| +SKILL.md | 36-37/50 |
| +SKILL2 | 36-37/50 |

**Why does CLAUDE.md outperform SKILL approaches?**

1. **Zero friction**: Docs index is passively available in context, no tool call needed
2. **Context position**: CLAUDE.md content appears early in context (higher attention)
3. **Model agency**: Model decides when/what to read based on the task

**What we learned about prompt phrasing:**

Tested different instructions:
- "Read docs FIRST before code" → Didn't help much
- "Explore codebase FIRST, then docs" → Similar results
- **Neutral "docs available for reference"** → Currently testing

The ordering instructions in prompts seem to matter less than the **mechanism** of how docs are made available (passive index vs active skill invocation).

## Conversation Transcript Analysis

Transcripts stored in `~/.claude/projects/<encoded-path>/*.jsonl`

Analyzing tool call sequences revealed:
```
CLAUDE.md pattern:
  Read(docs) → Glob → Read(code) → Write

SKILL pattern:
  Skill(invoke) → Read(code) → Read(docs) → Write
```

Even when SKILL2 reads docs, they arrive later in the reasoning process.

## Technical Implementation

**File: `lib/claude-code-runner.ts`**

Key functions:
- `runNextSkills()` - Creates CLAUDE.md with doc index
- `createNextjsSkill()` - Creates .claude/skills/nextjs-doc/SKILL.md
- `createNextjsSkill2()` - Creates both SKILL.md + CLAUDE.md mention
- `verifySkillUsage()` - Parses JSONL transcripts to check if skill was invoked

**Skill verification** reads the conversation JSONL and checks for:
- Skill tool invocation with `"skill": "nextjs-doc"`
- Read tool calls to `.mdx` doc files

## Repository Structure

```
next-evals-oss/
├── cli.ts                 # Main CLI entry point
├── lib/
│   └── claude-code-runner.ts  # Core eval runner
├── evals/
│   ├── 001-server-component/
│   │   ├── input/         # Starter Next.js project
│   │   ├── prompt.md      # Task description
│   │   └── output-*/      # Generated outputs (gitignored)
│   ├── 029-use-cache-directive/
│   └── ...
└── CLAUDE.md              # Instructions for running evals
```

## Open Questions

1. Does the **content** of docs matter, or just their **availability**?
2. Would embedding key patterns directly in CLAUDE.md outperform linking to doc files?
3. Can skill invocation overhead be eliminated while keeping on-demand flexibility?
4. How much of the 4-point gap between CLAUDE.md and SKILL approaches is reducible?

## Related

- Claude Code skills
- CLAUDE.md best practices
- AI agent evaluation
- Next.js App Router
- Documentation-driven development
