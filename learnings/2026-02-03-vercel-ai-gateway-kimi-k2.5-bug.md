# Vercel AI Gateway Bug: Kimi K2.5 Tool Call Failure

**Date:** 2026-02-03
**Project:** agent-eval (OpenCode integration)

## Summary

Kimi K2.5 model fails with `GatewayInternalServerError: Bad Request` after the first tool call when used with OpenCode through Vercel AI Gateway.

## Details

### Observed Behavior

1. Model receives prompt and starts processing
2. Model successfully executes one tool call (e.g., `bash: ls -la` or `glob`)
3. Tool completes successfully and returns output
4. When OpenCode sends the tool result back for the next turn, Gateway returns `Bad Request`

### Transcript Pattern

```
step_start → tool_use (succeeds) → step_finish → error
```

### Error Message

```json
{"type":"error","error":{"name":"UnknownError","data":{"message":"GatewayInternalServerError: Bad Request"}}}
```

### Models Tested

| Model | Result |
|-------|--------|
| `moonshotai/kimi-k2.5` | ❌ Gateway error after first tool |
| `moonshotai/kimi-k2` | ❌ Runs but doesn't complete task |
| `moonshotai/kimi-k2-thinking` | ✅ Works correctly |
| `anthropic/claude-sonnet-4` | ✅ Works correctly |

## Root Cause

The issue appears to be a Gateway-level compatibility problem specific to the `kimi-k2.5` model variant. The tool result format sent back by OpenCode may not match what K2.5 expects through the Gateway.

## Workaround

Use `moonshotai/kimi-k2-thinking` instead of `moonshotai/kimi-k2.5` for OpenCode tasks requiring tool use.

## Action Required

Report to Vercel AI Gateway team for investigation.
