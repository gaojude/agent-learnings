# Plan for Adding New Agent Runners to eval-framework

**Date:** 2026-01-26
**Project:** @judegao/eval (eval-framework)

## Context

The eval-framework currently only supports `claude-code` as an agent. To support additional agents (e.g., OpenAI Codex, Cursor, Aider, custom agents), the architecture needs to be refactored with a proper agent abstraction layer.

## Current Architecture Issues

### 1. Tight Coupling in Runner
The runner directly imports and calls `runAgent()` from `agent.ts`:
```typescript
// runner.ts:12
import { runAgent } from './agent.js';

// runner.ts:67
const agentResult = await runAgent(fixture.path, { ... });
```

### 2. Hardcoded Agent Type
The type system only allows one agent:
```typescript
// types.ts:8
export type AgentType = 'claude-code';
```

### 3. Zod Schema Uses Literal
Config validation only accepts one value:
```typescript
// config.ts:28
agent: z.literal('claude-code'),
```

### 4. Dry Run Missing Agent Info
CLI dry run output shows Model, Timeout, Early Exit but NOT the agent:
```typescript
// cli.ts:112
console.log(chalk.blue(`Model: ${config.model}, Timeout: ${config.timeout}s, Early Exit: ${config.earlyExit}`));
```

## Implementation Plan

### Step 1: Define Agent Interface

Create `src/lib/agents/types.ts`:
```typescript
import type { ModelTier, SetupFunction } from '../types.js';

/**
 * Common options for all agents.
 */
export interface AgentRunOptions {
  prompt: string;
  timeout: number;
  setup?: SetupFunction;
  scripts?: string[];
}

/**
 * Agent-specific options (extend per agent).
 */
export interface ClaudeCodeOptions extends AgentRunOptions {
  model: ModelTier;
  apiKey: string;
}

export interface OpenAICodexOptions extends AgentRunOptions {
  model: 'code-davinci-002' | 'gpt-4';
  apiKey: string;
}

/**
 * Common result structure for all agents.
 */
export interface AgentRunResult {
  success: boolean;
  output: string;
  transcript?: string;
  error?: string;
  duration: number;
  buildSuccess?: boolean;
  lintSuccess?: boolean;
  testSuccess?: boolean;
  buildOutput?: string;
  lintOutput?: string;
  testOutput?: string;
  sandboxId?: string;
  generatedFiles?: Record<string, string>;
}

/**
 * Agent interface that all agents must implement.
 */
export interface Agent {
  /** Unique identifier for the agent */
  name: string;

  /** Human-readable display name */
  displayName: string;

  /** Run the agent on a fixture */
  run(fixturePath: string, options: AgentRunOptions): Promise<AgentRunResult>;

  /** Validate agent-specific configuration */
  validateOptions?(options: unknown): void;

  /** Get agent-specific environment requirements */
  getRequiredEnvVars?(): string[];
}
```

### Step 2: Create Agent Registry

Create `src/lib/agents/registry.ts`:
```typescript
import type { Agent } from './types.js';
import type { AgentType } from '../types.js';

const agents = new Map<string, Agent>();

export function registerAgent(agent: Agent): void {
  agents.set(agent.name, agent);
}

export function getAgent(name: AgentType): Agent {
  const agent = agents.get(name);
  if (!agent) {
    const available = Array.from(agents.keys()).join(', ');
    throw new Error(`Unknown agent: ${name}. Available agents: ${available}`);
  }
  return agent;
}

export function listAgents(): Agent[] {
  return Array.from(agents.values());
}
```

### Step 3: Refactor Claude Code Agent

Move and refactor `src/lib/agent.ts` to `src/lib/agents/claude-code.ts`:
```typescript
import type { Agent, AgentRunResult, ClaudeCodeOptions } from './types.js';
import { SandboxManager, collectLocalFiles, splitTestFiles, verifyNoTestFiles } from '../sandbox.js';

export const claudeCodeAgent: Agent = {
  name: 'claude-code',
  displayName: 'Claude Code',

  async run(fixturePath: string, options: ClaudeCodeOptions): Promise<AgentRunResult> {
    // ... existing runAgent() implementation
  },

  validateOptions(options: unknown): void {
    if (!options || typeof options !== 'object') {
      throw new Error('Claude Code requires options object');
    }
    const opts = options as Record<string, unknown>;
    if (!opts.apiKey) {
      throw new Error('ANTHROPIC_API_KEY is required for Claude Code agent');
    }
  },

  getRequiredEnvVars(): string[] {
    return ['ANTHROPIC_API_KEY'];
  },
};
```

### Step 4: Create Agent Index with Auto-Registration

Create `src/lib/agents/index.ts`:
```typescript
import { registerAgent, getAgent, listAgents } from './registry.js';
import { claudeCodeAgent } from './claude-code.js';

// Auto-register built-in agents
registerAgent(claudeCodeAgent);

// Re-export
export { registerAgent, getAgent, listAgents };
export type { Agent, AgentRunOptions, AgentRunResult } from './types.js';
```

### Step 5: Update Type Definitions

Update `src/lib/types.ts`:
```typescript
/**
 * Supported AI agent types.
 * Add new agents here as union members.
 */
export type AgentType = 'claude-code' | 'openai-codex' | 'aider' | 'cursor';

/**
 * Agent-specific configuration.
 * Each agent can have its own model/options structure.
 */
export interface AgentConfig {
  'claude-code': {
    model?: 'opus' | 'sonnet' | 'haiku';
  };
  'openai-codex': {
    model?: 'code-davinci-002' | 'gpt-4';
  };
  // ... other agents
}
```

### Step 6: Update Config Schema

Update `src/lib/config.ts`:
```typescript
const experimentConfigSchema = z.object({
  agent: z.enum(['claude-code', 'openai-codex', 'aider', 'cursor']),
  // Make model optional at this level - agent validates its own model
  model: z.string().optional(),
  // ... rest unchanged
});
```

### Step 7: Update Runner to Use Registry

Update `src/lib/runner.ts`:
```typescript
import { getAgent } from './agents/index.js';

export async function runExperiment(options: RunExperimentOptions): Promise<ExperimentResults> {
  const { config, fixtures, apiKey, ... } = options;

  // Get the agent from registry
  const agent = getAgent(config.agent);

  // Validate agent-specific env vars
  const requiredEnvVars = agent.getRequiredEnvVars?.() ?? [];
  for (const envVar of requiredEnvVars) {
    if (!process.env[envVar]) {
      throw new Error(`${envVar} environment variable is required for ${agent.displayName}`);
    }
  }

  for (const fixture of fixtures) {
    // ...
    const agentResult = await agent.run(fixture.path, {
      prompt: fixture.prompt,
      model: config.model,
      timeout: config.timeout * 1000,
      apiKey,
      setup: config.setup,
      scripts: config.scripts,
    });
    // ...
  }
}
```

### Step 8: Update CLI Dry Run Output

Update `src/cli.ts` line 112:
```typescript
console.log(chalk.blue(`Agent: ${config.agent}, Model: ${config.model}, Timeout: ${config.timeout}s, Early Exit: ${config.earlyExit}`));
```

### Step 9: Add Plugin Support (Future)

For user-defined agents, add plugin loading:
```typescript
// src/lib/agents/plugin.ts
export async function loadAgentPlugin(path: string): Promise<void> {
  const module = await import(path);
  if (module.default && typeof module.default.run === 'function') {
    registerAgent(module.default);
  }
}

// In experiment config
export interface ExperimentConfig {
  // ...
  agentPlugins?: string[]; // Paths to custom agent modules
}
```

## File Structure After Refactoring

```
src/lib/
  agents/
    index.ts          # Auto-registration and exports
    registry.ts       # Agent registry (Map-based)
    types.ts          # Agent interface and common types
    claude-code.ts    # Claude Code implementation
    openai-codex.ts   # OpenAI Codex implementation (future)
    aider.ts          # Aider implementation (future)
    plugin.ts         # Plugin loading for custom agents
  types.ts            # Updated AgentType union
  config.ts           # Updated schema with agent enum
  runner.ts           # Uses getAgent() from registry
  ...
```

## Migration Checklist

1. [ ] Create `src/lib/agents/` directory structure
2. [ ] Define `Agent` interface in `agents/types.ts`
3. [ ] Create registry in `agents/registry.ts`
4. [ ] Refactor `agent.ts` to `agents/claude-code.ts`
5. [ ] Create `agents/index.ts` with auto-registration
6. [ ] Update `AgentType` union in `types.ts`
7. [ ] Update Zod schema in `config.ts`
8. [ ] Update runner to use registry
9. [ ] Update CLI dry run output to show agent
10. [ ] Add tests for registry and new agent interface
11. [ ] Update README with new agent documentation
12. [ ] Bump version

## Adding a New Agent (Example: Aider)

Once the architecture is in place, adding a new agent requires:

1. Create `src/lib/agents/aider.ts`:
```typescript
import type { Agent, AgentRunResult, AgentRunOptions } from './types.js';

export const aiderAgent: Agent = {
  name: 'aider',
  displayName: 'Aider',

  async run(fixturePath: string, options: AgentRunOptions): Promise<AgentRunResult> {
    // 1. Create sandbox
    // 2. Upload files
    // 3. Run aider CLI with prompt
    // 4. Run validation
    // 5. Return results
  },

  getRequiredEnvVars(): string[] {
    return ['OPENAI_API_KEY']; // or ANTHROPIC_API_KEY depending on backend
  },
};
```

2. Register in `agents/index.ts`:
```typescript
import { aiderAgent } from './aider.js';
registerAgent(aiderAgent);
```

3. Add to `AgentType` union:
```typescript
export type AgentType = 'claude-code' | 'openai-codex' | 'aider';
```

4. Update config schema enum.

## Key Design Decisions

1. **Registry Pattern**: Central registry allows runtime agent lookup without hardcoding imports
2. **Interface-Based**: All agents implement same interface for consistency
3. **Self-Describing**: Agents declare their required env vars and display names
4. **Plugin Support**: Future path for user-defined agents without forking
5. **Backwards Compatible**: Existing configs with `agent: 'claude-code'` continue to work
