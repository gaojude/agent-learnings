# next-skills: Direct Docs Bundling vs Lazy Loading

**Date:** 2026-01-17
**Project:** next-skills

## Context

The next-skills CLI installs Next.js documentation as an "Agent Skill" for AI coding agents (Claude Code, Cursor, etc.). The skill consists of a `SKILL.md` file and associated documentation files.

## The Decision

We tried two approaches for how documentation gets delivered to agents:

### Approach 1: Lazy Loading (reverted)
- `SKILL.md` contained instructions telling the agent to run `npx @judegao/next-skills pull`
- Docs were fetched on-demand to a `/tmp` directory
- Agent had to run a command before accessing docs

### Approach 2: Direct Bundling (preferred)
- Docs are cloned directly into the skill folder during installation (e.g., `.claude/skills/nextjs-doc/docs/`)
- `SKILL.md` contains a documentation index with relative links to local files (e.g., `./docs/01-app/...`)
- Agent can immediately access docs without extra steps

## Why Direct Bundling is Better

1. **Zero friction** - Agent can read docs immediately without running commands
2. **Works offline** - Docs are already present after installation
3. **Simpler mental model** - One install step, everything is ready
4. **Better for SKILL.md format** - The index with local file links is the intended pattern for skills

## Key Files Changed

- `src/generator/template.ts` - Generates `SKILL.md` with doc index (takes `sections: DocSection[]`)
- `src/generator/index.ts` - `generateSkillWithClone()` clones docs and builds section tree
- `src/installer/index.ts` - Calls `generateSkillWithClone()` with a `docsDir` path

## How to Revert Between Approaches

To revert from lazy loading to direct bundling, restore the `sections` parameter to `SkillTemplateData` and ensure `generateSkillWithClone()` passes the built doc tree to `generateSkillMd()`.
