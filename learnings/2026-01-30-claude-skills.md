# Claude Skills

**Date:** 2026-01-30
**Context:** Next.js codebase - understanding Claude Code's skill system

## What are Claude Skills?

Claude skills are specialized tools that extend Claude Code's capabilities through structured commands. They provide domain-specific knowledge and workflows that can be invoked during conversations.

## Key Characteristics

### Invocation Methods
- Using the `Skill` tool directly
- User references with slash commands (e.g., `/commit`, `/review-pr`)
- Execute within the main conversation context
- Support both simple names and fully qualified names (e.g., `ms-office-suite:pdf`)

### Behavior Rules
- Listed in system-reminder messages in the conversation
- **Must be invoked BEFORE generating other responses** when they match user intent
- Never mention a skill without actually calling the Skill tool
- Don't invoke skills that are already running
- When loaded, appear as `<command-name>` tags in the conversation

## Available Skills (Examples)

- **keybindings-help**: Customize keyboard shortcuts
- **remember**: Save learnings to agent-learnings repo
- **update-docs**: Guided workflow for updating Next.js documentation
- **pr-status**: Analyze PR CI failures and reviews

## Invocation Examples

```typescript
// Simple invocation without arguments
skill: "commit"

// With arguments (e.g., PR number)
skill: "review-pr", args: "123"

// Fully qualified name (package-based skill)
skill: "ms-office-suite:pdf"
```

## Important Distinctions

**Not Skills:** Built-in CLI commands like `/help`, `/clear` are NOT invoked through the Skill tool. These are native CLI commands handled differently.

## Integration Pattern

Skills can be:
- **Built-in**: Part of Claude Code core functionality
- **Package-based**: Installed from external packages with namespaced names
- **Custom**: User-defined skills in specific directories
