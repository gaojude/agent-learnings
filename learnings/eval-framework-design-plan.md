# Eval Framework Design Plan

**Date**: 2026-01-26
**Project**: @anthropic/eval-framework
**Context**: Designing a zero-config, developer-friendly eval framework for testing AI coding agents across any frontend framework.

---

## Problem Statement

Build an eval framework that allows users to test AI coding agents (like Claude Code CLI) with:
- Minimal configuration (zero-config ideal)
- Support for any frontend framework (React, Vue, Svelte, Next.js, etc.)
- Isolated sandbox execution
- Built-in reporting
- Proper cleanup on interruption

---

## Core Design Principles

1. **Zero-config for common cases** - Convention over configuration (like Vercel/Next.js)
2. **Framework-agnostic** - Works with any frontend framework via `vercel build`
3. **Sandbox-agnostic** - Pluggable sandbox providers (local, Docker, Vercel)
4. **Agent-agnostic** - Support different AI agents (Claude Code, custom agents)
5. **Simple primitives** - `exec` is the core, everything builds on top

---

## File Structure Convention

```
evals/
  button/
    prompt.md              # The prompt sent to the agent (required)
    fixture/               # Starting codebase (required)
      package.json         # Defines build/test commands
      app/
        page.tsx
    button.test.ts         # Validation tests (withheld until after agent runs)

  form/
    prompt.md
    fixture/
    form.test.ts

eval.config.ts             # Optional global config
```

---

## CLI Usage

### Basic Commands

```bash
# Run all evals (zero-config)
npx eval

# Run specific eval
npx eval button

# Run with specific sandbox
npx eval --sandbox docker
npx eval --sandbox vercel
npx eval --sandbox local      # Shows big warning about no isolation

# JSON output for programmatic use
npx eval button --json
npx eval button --json --no-transcript

# Specify agent
npx eval --agent claude-code    # default
npx eval --agent ./my-agent.ts  # custom
```

### A/B Comparison (via pre-hooks)

```bash
# Compare baseline vs with-skills
npx eval --compare baseline "npx @judegao/next-skills"

# Multiple runs for statistical significance
npx eval --runs 5
```

---

## Output Examples

### Human-readable (default)

```
$ npx eval

  Detecting sandbox...
  ✓ Docker found

  button    ✓ build  ✓ lint  ✓ tests  (23s)
  form      ✓ build  ✓ lint  ✗ tests  (19s)

  1/2 passed
```

### Local sandbox warning

```
$ npx eval

  Detecting sandbox...
  ✗ Docker not found

  ┌─────────────────────────────────────────────────────────────────┐
  │  ⚠️  WARNING: Running without isolation                         │
  │                                                                 │
  │  No sandbox available. Evals will run directly on your machine. │
  │  The AI agent will have full access to your file system.        │
  │                                                                 │
  │  For isolation, install Docker or set VERCEL_TOKEN.             │
  │                                                                 │
  │  Press Enter to continue, or Ctrl+C to abort...                 │
  └─────────────────────────────────────────────────────────────────┘
```

### JSON output

```json
{
  "eval": "button",
  "sandbox": "docker",
  "success": true,
  "duration": 23000,
  "build": { "success": true, "output": "..." },
  "lint": { "success": true, "output": "..." },
  "tests": { "success": true, "output": "..." },
  "agentOutput": "Created Button.tsx with...",
  "transcript": [
    { "role": "user", "content": "Create an accessible button..." },
    { "role": "assistant", "content": "I'll create...", "toolCalls": [...] }
  ]
}
```

---

## Architecture

### Layers

```
┌─────────────────────────────────────────┐
│  CLI (npx eval)                         │  ← Zero-config entry point
├─────────────────────────────────────────┤
│  EvalRunner                             │  ← Orchestration, file discovery
├─────────────────────────────────────────┤
│  Agent (claude-code, custom)            │  ← AI agent abstraction
├─────────────────────────────────────────┤
│  Sandbox (local, docker, vercel)        │  ← Execution environment
└─────────────────────────────────────────┘
```

### Sandbox Interface

```typescript
interface Sandbox {
  exec(cmd: string): Promise<{ stdout: string; stderr: string; exitCode: number }>;
  writeFiles(files: { path: string; content: Buffer }[]): Promise<void>;
  readFile(path: string): Promise<string>;
  dispose(): Promise<void>;
}
```

### Agent Interface

```typescript
interface Agent {
  name: string;
  install(sandbox: Sandbox): Promise<void>;
  run(sandbox: Sandbox, prompt: string): Promise<string>;  // returns stdout
  getTranscript(sandbox: Sandbox): Promise<string>;
}
```

### Claude Code Agent Implementation

```typescript
const claudeCode: Agent = {
  name: 'claude-code',

  install: async (sandbox) => {
    await sandbox.exec('npm install -g @anthropic-ai/claude-code');
  },

  run: async (sandbox, prompt) => {
    const result = await sandbox.exec(
      `ANTHROPIC_API_KEY=${key} claude --print --dangerously-skip-permissions "${prompt}"`
    );
    return result.stdout;
  },

  getTranscript: async (sandbox) => {
    const { stdout } = await sandbox.exec(
      'cat ~/.claude/projects/*/transcript.jsonl 2>/dev/null || echo "[]"'
    );
    return stdout;
  },
};
```

---

## Sandbox Providers

| Sandbox | Pros | Cons | Requires |
|---------|------|------|----------|
| `local` | Zero deps, fast | No isolation ⚠️ | Nothing |
| `docker` | Isolated, free | Needs Docker | Docker installed |
| `vercel` | Fast, cloud, parallel | Needs account | `VERCEL_TOKEN` |

### Detection Priority

```typescript
function detectSandbox(): SandboxProvider {
  if (process.env.VERCEL_TOKEN) {
    return vercel();
  }
  if (isDockerRunning()) {
    return docker();
  }
  return local(); // Shows warning, requires confirmation
}
```

---

## Execution Flow

1. **Discover evals** - Find all `evals/*/` directories
2. **For each eval**:
   - Create sandbox with `fixture/` files
   - Run `npm install`
   - Run agent with `prompt.md` content
   - Inject `*.test.ts` files (withheld during agent run)
   - Run `vercel build` (works for any framework)
   - Run `npm run lint` (if exists)
   - Run `npm test`
   - Collect results + transcript
3. **Report results**
4. **Cleanup sandboxes**

---

## Cleanup on Interrupt (Critical!)

Must handle Ctrl+C and cleanup cloud sandboxes to prevent billing:

```typescript
const activeSandboxes = new Set<Sandbox>();

async function cleanup() {
  if (activeSandboxes.size === 0) return;

  console.log('\n\n  Cleaning up sandboxes...');

  await Promise.all(
    [...activeSandboxes].map(async (sandbox) => {
      try {
        await sandbox.dispose();
        console.log('  ✓ Sandbox terminated');
      } catch (e) {
        console.log('  ✗ Failed to terminate sandbox');
      }
    })
  );

  activeSandboxes.clear();
}

process.on('SIGINT', async () => {
  await cleanup();
  process.exit(130);
});

process.on('SIGTERM', async () => {
  await cleanup();
  process.exit(143);
});

process.on('uncaughtException', async (err) => {
  console.error(err);
  await cleanup();
  process.exit(1);
});
```

---

## Programmatic API (Advanced Usage)

For users who need custom validation beyond build/lint/test:

```typescript
import { EvalSandbox, claudeCode } from '@anthropic/eval-framework';

const sandbox = await EvalSandbox.create({
  fixture: './my-app',
  sandbox: 'docker',
});

await claudeCode.run(sandbox, 'Create a button');

// Custom probing via exec
const { exitCode } = await sandbox.exec('grep -r "aria-label" app/');
const { stdout } = await sandbox.exec('cat app/components/Button.tsx');

// Get agent transcript
const transcript = await claudeCode.getTranscript(sandbox);

await sandbox.dispose();
```

---

## Configuration (Optional)

For cases that need customization:

```typescript
// eval.config.ts (global)
export default {
  sandbox: 'docker',
  timeout: 300000,
};

// evals/button/eval.config.ts (per-eval override)
export default {
  build: 'npm run build',      // override build command
  test: 'npm test',            // override test command
  timeout: 600000,
  preHook: async (sandbox) => {
    await sandbox.exec('npx @judegao/next-skills --agent claude');
  },
};
```

---

## V1 Scope

- **Sandboxes**: local (with warning), docker, vercel
- **Agents**: claude-code built-in, custom agent support via interface
- **Reporting**: single run output (human-readable + JSON)
- **Cleanup**: always cleanup on any exit signal
- **Aggregation**: deferred to users (pipe JSON to their own tools)

---

## Future Considerations (Not V1)

- Built-in aggregation and A/B comparison reporting
- Statistical significance testing (p-values)
- Flakiness detection mode
- More sandbox providers (E2B, etc.)
- More built-in agents (Cursor, Copilot, etc.)
- Web UI for viewing results
- CI/CD integrations
