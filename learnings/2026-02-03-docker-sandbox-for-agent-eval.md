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

### CLI Output - Sandbox Backend Logging

**Important**: Since the default is automatic, the CLI must print which sandbox backend is being used so users understand what's happening:

```
Loading config from experiments/cc.ts...
Discovering evals in /path/to/evals...

Found 3 valid fixture(s), will run 3:
  - fixture-a
  - fixture-b
  - fixture-c

Running 3 eval(s) x 1 run(s) = 3 total runs
Agent: claude-code, Model: opus, Timeout: 300s, Early Exit: true
Sandbox: docker (auto-detected)    ← NEW LINE

Starting experiment...
```

Possible sandbox messages:
- `Sandbox: docker (auto-detected)` - No Vercel token found, using Docker
- `Sandbox: vercel (auto-detected)` - Vercel token found, using Vercel
- `Sandbox: docker (explicit)` - User set SANDBOX_BACKEND=docker
- `Sandbox: vercel (explicit)` - User set SANDBOX_BACKEND=vercel

## Files to Modify

| File | Change |
|------|--------|
| `src/lib/sandbox.ts` | Add `createSandbox()` factory, `resolveBackend()`, keep `SandboxManager` as `VercelSandboxManager` |
| `src/lib/docker-sandbox.ts` | **New file** - `DockerSandboxManager` implementation |
| `src/lib/types.ts` | Add `SandboxBackend` type |
| `src/lib/agents/claude-code.ts` | Use `createSandbox()` instead of `SandboxManager.create()` |
| `src/lib/agents/codex.ts` | Same |
| `src/cli.ts` | Print sandbox backend info before starting experiment |
| `src/index.ts` | Export `createSandbox`, `DockerSandboxManager`, `resolveBackend` |
| `package.json` | Add `dockerode` dependency |

## Docker Implementation Details

### What is `dockerode`?

`dockerode` is the most popular Node.js library for interacting with the Docker Engine API. It provides a programmatic way to:

- Manage containers (create, start, stop, remove)
- Execute commands inside containers
- Copy files into/out of containers
- Pull images
- Stream logs

**Example usage:**
```typescript
import Docker from 'dockerode';

const docker = new Docker();  // Connects to local Docker daemon via /var/run/docker.sock

// Create and start a container
const container = await docker.createContainer({
  Image: 'node:24-slim',
  Cmd: ['sleep', 'infinity'],  // Keep container alive
  WorkingDir: '/workspace',
  Tty: true,
});
await container.start();

// Execute a command inside container
const exec = await container.exec({
  Cmd: ['npm', 'install'],
  AttachStdout: true,
  AttachStderr: true,
  WorkingDir: '/workspace',
});
const stream = await exec.start({ hijack: true, stdin: false });
// Collect stdout/stderr from stream...

// Copy files into container (using tar archive)
import tar from 'tar-stream';
const pack = tar.pack();
pack.entry({ name: 'package.json' }, '{"name": "test"}');
pack.finalize();
await container.putArchive(pack, { path: '/workspace' });

// Read file from container
const readExec = await container.exec({
  Cmd: ['cat', '/workspace/package.json'],
  AttachStdout: true,
});
// ...

// Stop and remove
await container.stop();
await container.remove();
```

### Why Node.js `-slim` images?

Docker Hub provides official Node.js images in different variants:

| Image | Size | Contents |
|-------|------|----------|
| `node:24` | ~1GB | Full Debian with build tools (gcc, make, python) |
| `node:24-slim` | ~200MB | Minimal Debian, just Node.js runtime |
| `node:24-alpine` | ~150MB | Alpine Linux (can have glibc compatibility issues) |

**We use `-slim` because:**
- **Smaller download** - ~200MB vs ~1GB for full image
- **Faster startup** - Less to load into memory
- **Sufficient for most packages** - Has `npm`, can install pure-JS packages
- **Better compatibility than Alpine** - Uses glibc like most dev machines

**Note**: If a fixture needs native compilation (node-gyp), it may fail on slim. We could add a `runtime: 'node24-full'` option later if needed.

### Image Pull Strategy

The implementation will:
1. Check if the required image exists locally (`docker.getImage().inspect()`)
2. If not present, pull it with progress output:
   ```
   Pulling node:24-slim... [=====>    ] 45%
   ```
3. Cache for future runs (Docker handles this automatically)

### Container Lifecycle

```
createSandbox() called
    │
    ▼
┌─────────────────────────────────────┐
│  docker.createContainer({           │
│    Image: 'node:24-slim',           │
│    Cmd: ['sleep', 'infinity'],      │
│    WorkingDir: '/workspace',        │
│    HostConfig: {                    │
│      AutoRemove: true,              │
│      Memory: 2 * 1024 * 1024 * 1024 │  ← 2GB limit (optional)
│    }                                │
│  })                                 │
└─────────────────────────────────────┘
    │
    ▼
container.start()
    │
    ▼
uploadFiles() → container.putArchive(tarStream, {path: '/workspace'})
    │
    ▼
runCommand() → container.exec({Cmd: [...]}) → stream stdout/stderr
    │
    ▼
readFile() → container.exec({Cmd: ['cat', path]})
    │
    ▼
stop() → container.stop() → container auto-removed
```

## Docker Installation for Users

```bash
# macOS
brew install --cask docker
# Then open Docker Desktop to start the daemon

# Ubuntu/Debian
sudo apt-get update
sudo apt-get install docker.io
sudo systemctl start docker
sudo usermod -aG docker $USER  # to run without sudo (requires re-login)

# Verify installation
docker --version
docker run hello-world
```

## Dependencies to Add

```bash
npm install dockerode
npm install --save-dev @types/dockerode

# tar-stream for creating tar archives to upload files
npm install tar-stream
npm install --save-dev @types/tar-stream
```

## Usage After Implementation

```bash
# No Vercel token → automatically uses Docker
agent-eval my-experiment
# Output: "Sandbox: docker (auto-detected)"

# Explicit Docker
SANDBOX_BACKEND=docker agent-eval my-experiment
# Output: "Sandbox: docker (explicit)"

# Explicit Vercel (requires token)
SANDBOX_BACKEND=vercel VERCEL_TOKEN=xxx agent-eval my-experiment
# Output: "Sandbox: vercel (explicit)"
```
