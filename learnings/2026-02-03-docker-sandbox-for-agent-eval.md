# Adding Docker Sandbox Support to agent-eval

**Date**: 2026-02-03
**Project**: agent-eval (Vercel agent testing framework)

## Context

The agent-eval framework currently only supports Vercel Sandbox for running isolated evals. Users want Docker as an alternative for local development without Vercel credentials.

## Current Architecture

The codebase has a clean abstraction:

1. **`Sandbox` interface** (`src/lib/types.ts:29-42`) - The contract agents use:
   ```typescript
   interface Sandbox {
     runCommand(command, args?, options?): Promise<CommandResult>
     readFile(path): Promise<string>
     writeFiles(files: Record<string, string>): Promise<void>
     getWorkingDirectory(): string
   }
   ```

2. **`SandboxManager`** (`src/lib/sandbox.ts:71`) - Implements `Sandbox` using `@vercel/sandbox`

3. **Agents are backend-agnostic** - They call `SandboxManager.create()` and use the interface

## Proposed Design

### API Design

```typescript
// Environment variables (priority order):
// 1. SANDBOX_BACKEND=vercel|docker  (explicit override)
// 2. If VERCEL_TOKEN or VERCEL_OIDC_TOKEN exists → use vercel
// 3. Default → docker

// Updated SandboxOptions
interface SandboxOptions {
  timeout?: number;
  runtime?: 'node20' | 'node24';
  backend?: 'vercel' | 'docker' | 'auto';  // default: 'auto'
}

// Factory function (new)
async function createSandbox(options?: SandboxOptions): Promise<Sandbox>
```

### Auto-detection Logic

```typescript
function resolveBackend(options?: SandboxOptions): 'vercel' | 'docker' {
  // Explicit override
  if (options?.backend && options.backend !== 'auto') {
    return options.backend;
  }

  // Env var override
  const envBackend = process.env.SANDBOX_BACKEND;
  if (envBackend === 'vercel' || envBackend === 'docker') {
    return envBackend;
  }

  // Auto-detect: Vercel if token present, else Docker
  if (process.env.VERCEL_TOKEN || process.env.VERCEL_OIDC_TOKEN) {
    return 'vercel';
  }

  return 'docker';
}
```

## Files to Modify

| File | Change |
|------|--------|
| `src/lib/sandbox.ts` | Add `createSandbox()` factory, keep `SandboxManager` as `VercelSandboxManager` |
| `src/lib/docker-sandbox.ts` | **New file** - `DockerSandboxManager` implementation |
| `src/lib/types.ts` | Add `SandboxBackend` type |
| `src/lib/agents/claude-code.ts` | Use `createSandbox()` instead of `SandboxManager.create()` |
| `src/lib/agents/codex.ts` | Same |
| `src/index.ts` | Export `createSandbox`, `DockerSandboxManager` |
| `package.json` | Add `dockerode` dependency |

## Docker Implementation Details

The Docker sandbox will:

1. **Use `dockerode`** - The standard Node.js Docker API client
2. **Pull/use Node.js image** - `node:24-slim` or `node:20-slim` based on runtime option
3. **Create container** with:
   - Working directory: `/workspace`
   - Auto-remove on stop
   - Resource limits (optional)
4. **Copy files** via `putArchive()` (tar stream)
5. **Execute commands** via `exec()` API
6. **Read files** via `exec('cat', [path])`
7. **Clean up** container on `stop()`

## Docker Installation for Users

```bash
# macOS
brew install --cask docker
# Then open Docker Desktop to start the daemon

# Ubuntu/Debian
sudo apt-get update
sudo apt-get install docker.io
sudo systemctl start docker
sudo usermod -aG docker $USER  # to run without sudo

# Verify
docker --version
docker run hello-world
```

## Dependencies to Add

```bash
npm install dockerode
npm install --save-dev @types/dockerode
```

## Usage After Implementation

```bash
# No Vercel token → automatically uses Docker
agent-eval my-experiment

# Explicit Docker
SANDBOX_BACKEND=docker agent-eval my-experiment

# Explicit Vercel (requires token)
SANDBOX_BACKEND=vercel VERCEL_TOKEN=xxx agent-eval my-experiment
```
