# Debugging Docs Bias in Eval Regressions

**Date:** 2026-01-16

**Project:** next-evals-oss

## Summary

When debugging eval regressions where +Docs fails but CC passes, documentation examples can bias the agent toward specific patterns that don't match test expectations.

## Case Study: 012-parallel-routes

The prompt asked for a "dashboard layout with parallel routes". Combined with Next.js docs showing `app/dashboard/layout.tsx` examples, the +Docs agent consistently created:
- `app/dashboard/@analytics` (wrong)

Instead of what the test expected:
- `app/@analytics` (correct)

The CC agent (without docs) was more flexible and pivoted to match test expectations on retry.

## Key Debugging Steps

1. **Compare output directory structures** - Check what files were created and where
2. **Read conversation .jsonl files** - Located at `~/.claude/projects/` with path-encoded directory names
3. **Identify docs influence** - Find which documentation was read and how it shaped decisions
4. **Check test awareness** - Verify if test expectations were considered by the agent

## Pattern: Literal Doc Interpretation

Agents may copy documentation examples verbatim without adapting to specific test requirements. When docs show nested structures (like `/dashboard`), agents follow that pattern even when simpler root-level solutions are correct.
