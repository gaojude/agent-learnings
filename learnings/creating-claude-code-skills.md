# Creating Claude Code Skills

**Date:** 2026-01-16
**Project:** next-evals-oss

## Summary

Claude Code skills are custom slash commands defined as SKILL.md files that extend Claude's capabilities with project-specific workflows.

## Location

Skills are stored in:
```
.claude/skills/<skill-name>/SKILL.md
```

## File Structure

Each skill is a single `SKILL.md` file with YAML frontmatter and Markdown body:

```markdown
---
name: <skill-name>
allowed-tools: <comma-separated list of tools>
description: <brief description shown in help>
---

## <Skill Title>

<Description and purpose>

### Usage

`/<skill-name>` - Basic usage
`/<skill-name> <args>` - With arguments

### What this skill helps with

- <Use case 1>
- <Use case 2>

### Your task

#### 1. <First step>
<Instructions>

#### 2. <Second step>
<Instructions>

### <Additional sections as needed>
```

## YAML Frontmatter Fields

| Field | Required | Description |
|-------|----------|-------------|
| `name` | Yes | Skill identifier, used as `/<name>` command |
| `allowed-tools` | Yes | Tools the skill can access |
| `description` | Yes | Brief description for skill listing |

## Tool Restrictions

The `allowed-tools` field controls what tools the skill can use:

- **Full tool access:** `Read, Write, Glob, Grep, Bash, Task`
- **Restricted Bash:** `Bash(git:*), Bash(ls:*)` - only specific subcommands
- **Common patterns:**
  - Read-only exploration: `Read, Glob, Grep, Bash(ls:*), Bash(find:*)`
  - Git operations: `Bash(git:*)`
  - File creation: `Read, Write, Glob`

## Best Practices

1. **Be specific about tools** - Only grant tools the skill actually needs
2. **Structure tasks clearly** - Use numbered steps for the agent to follow
3. **Include examples** - Show expected output formats and patterns
4. **Document common patterns** - Help the agent recognize failure modes
5. **Provide context** - Explain why certain approaches work better

## Example: Minimal Skill

```markdown
---
name: greet
allowed-tools: Read
description: Greet the user with project info
---

## Greet Skill

### Your task

1. Read the project's package.json or similar config
2. Greet the user with the project name and version
3. List any notable features or recent changes
```

## Invoking Skills

Users invoke skills via slash commands:
- `/remember` - invoke the remember skill
- `/debug-eval 040-intercepting-routes` - invoke with arguments

The skill system expands the SKILL.md content into the conversation as instructions for the agent to follow.
