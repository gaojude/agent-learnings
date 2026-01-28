# Debugging Eval Failures in next-evals-oss

## Summary
How to debug why an eval failed by examining the saved JSON result file.

## Context
When running evals with `bun cli.ts`, results are saved to `eval-results/` even when the output directories are cleaned up. This allows post-mortem debugging without re-running with `--debug`.

## Details

### 1. Run the eval

```bash
bun cli.ts --eval agent-039-indirect-proxy --claude-code --pre-hook "npx @judegao/next-agents-md --md CLAUDE.md" --retries 4 --dry --verbose
```

Key flags:
- `--retries N` - Run N+1 concurrent attempts to reduce model randomness
- `--pre-hook` - Command to run before Claude Code (e.g., inject docs)
- `--dry` - Run locally without uploading to Braintrust
- `--verbose` - Show detailed logs
- `--debug` - Keep output folders (optional, not needed for JSON inspection)

### 2. Find the result file

Results are saved to:
```
eval-results/eval-result-<eval-name>-<timestamp>.json
```

### 3. Inspect the JSON structure

```bash
# Check top-level keys
cat eval-results/eval-result-*.json | jq '.result | keys'

# Keys available:
# - buildOutput, buildSuccess
# - lintOutput, lintSuccess
# - testOutput, testSuccess
# - generatedFiles
# - claudeMdContent
# - output (full Claude Code output)
# - duration, timestamp, sandboxId
```

### 4. Debug test failures

```bash
# Get test output to see which tests failed
cat eval-results/eval-result-*.json | jq -r '.result.testOutput'
```

### 5. Check what files the agent created

```bash
# List all generated file paths
cat eval-results/eval-result-*.json | jq -r '.result.generatedFiles | keys[]'

# Filter for specific files
cat eval-results/eval-result-*.json | jq -r '.result.generatedFiles | keys[]' | grep -E "proxy|middleware"

# Read content of a specific generated file
cat eval-results/eval-result-*.json | jq -r '.result.generatedFiles["/vercel/sandbox/middleware.ts"]'
```

### 6. Verify pre-hook worked

```bash
# Check if CLAUDE.md was generated correctly
cat eval-results/eval-result-*.json | jq -r '.result.claudeMdContent' | head -20
```

## Example Debug Session

```bash
# Run eval with retries
bun cli.ts --eval agent-039-indirect-proxy --claude-code --retries 4 --dry

# Output shows: ✅✅❌ (Build/Lint pass, Test fails)

# Check test output
cat eval-results/eval-result-agent-039-indirect-proxy-*.json | jq -r '.result.testOutput'
# Shows: "expected false to be true" for proxy.ts existence check

# Check what files were created
cat eval-results/eval-result-agent-039-indirect-proxy-*.json | jq -r '.result.generatedFiles | keys[]' | grep -E "proxy|middleware"
# Shows: /vercel/sandbox/middleware.ts (wrong! should be proxy.ts)

# Inspect the wrong file
cat eval-results/eval-result-agent-039-indirect-proxy-*.json | jq -r '.result.generatedFiles["/vercel/sandbox/middleware.ts"]'
# Shows agent created deprecated middleware.ts instead of proxy.ts
```

## Related
- next-evals-oss CLI: `bun cli.ts --help`
- Eval structure: `evals/<eval-name>/input/`, `prompt.md`, `README.md`
- Tags: #debugging #evals #next-evals-oss #jq
