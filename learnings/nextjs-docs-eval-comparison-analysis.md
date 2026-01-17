# Next.js Docs Eval Comparison Analysis

## Summary

Analysis of eval results comparing Claude Code baseline vs CLAUDE.md augmentation vs SKILL.md (doc-pulling) approaches for Next.js tasks, revealing modest gains from documentation but persistent failure patterns.

## Context

When evaluating whether providing Next.js documentation improves Claude Code's ability to complete Next.js-related coding tasks. This analysis covers 50 evals testing various Next.js patterns including App Router, Server Components, AI SDK integration, and routing patterns.

## Results Overview

| Condition | Pass Rate | Delta |
|-----------|-----------|-------|
| Claude Code (baseline) | 31/50 (62%) | - |
| + CLAUDE.md | 34/50 (68%) | +3 |
| + SKILL.md | 34/50 (68%) | +3 |

### Legend
- `✅✅✅` = Build/Lint/Test all pass
- `🔄✅` = Retry passed
- `🔄❌` = Retry failed
- `📚` = Docs pulled & read
- `📥` = Pulled only
- `⚠️` = Skill not used

## Key Insights

### 1. CLAUDE.md Provides Modest Improvement (+3 tests)

**Cases where CLAUDE.md helps:**
- `007-client-metadata`: ✅✅❌ → ✅✅✅
- `015-server-actions-exports`: ✅✅❌ → ✅✅✅
- `020-no-use-effect`: ✅✅❌ → ✅✅✅
- `029-use-cache-directive`: ❌✅✅ → ✅✅✅
- `033-ai-sdk-v4-model-specification-function`: ❌✅❌ → ❌✅✅ (partial)
- `039-parallel-routes`: ✅✅❌ → ✅✅✅

**Cases where CLAUDE.md causes regressions:**
- `014-server-routing`: ✅✅✅ → ✅✅❌
- `049-revalidation`: ✅✅✅ → ✅✅❌

### 2. SKILL.md Not Providing Expected Lift

Despite pulling and reading docs (📚) in most cases, SKILL.md ties with CLAUDE.md at 34/50. The ⚠️ markers show the skill wasn't even triggered in ~15 cases, missing opportunities to help.

### 3. Persistently Failing Evals (All 3 Conditions Fail)

**Next.js routing patterns:**
- `012-parallel-routes`
- `013-pathname-server`
- `016-client-cookies`
- `040-intercepting-routes`

**AI SDK evals:**
- `034-ai-sdk-render-visual-info`
- `035-ai-sdk-call-tools`
- `037-ai-sdk-embed-text`

### 4. Retries Rarely Help

The `🔄❌` pattern is far more common than `🔄✅`, indicating failures are usually systematic, not flaky. This suggests the issues are fundamental understanding gaps rather than random errors.

### 5. AI SDK Evals Are Particularly Challenging

Evals 031-038 show weak performance across all conditions, with most showing ❌ on build or test. This area has the weakest coverage.

## Full Results Table

```
| Eval                           | Claude Code      | + CLAUDE.md      | + SKILL.md       |
|--------------------------------|------------------|------------------|------------------|
| 000-app-router-migration-simple | ✅✅❌ 🔄❌          | ✅✅❌ 🔄❌          | ✅✅❌ 🔄❌ ⚠️       |
| 001-server-component           | ✅✅✅              | ✅✅✅ 🔄✅          | ✅✅✅ ⚠️           |
| 002-client-component           | ✅✅✅              | ✅✅✅              | ✅✅✅ ⚠️           |
| 003-cookies                    | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 004-search-params              | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 005-react-use-api              | ✅✅✅              | ✅✅✅              | ✅✅✅ ⚠️           |
| 006-server-metadata            | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 007-client-metadata            | ✅✅❌ 🔄❌          | ✅✅✅              | ✅✅✅ 📚           |
| 008-generate-static-params     | ✅✅✅              | ✅✅✅              | ✅✅✅ ⚠️           |
| 009-og-images                  | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 010-route-handlers             | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 011-client-server-form         | ✅✅✅              | ✅✅✅ 🔄✅          | ✅✅✅ 📚           |
| 012-parallel-routes            | ✅✅❌ 🔄❌          | ✅✅❌ 🔄❌          | ✅✅❌ 🔄❌ 📚       |
| 013-pathname-server            | ✅✅❌ 🔄❌          | ✅✅❌ 🔄❌          | ✅✅❌ 🔄❌ 📚       |
| 014-server-routing             | ✅✅✅              | ✅✅❌ 🔄❌          | ✅✅✅ 📚           |
| 015-server-actions-exports     | ✅✅❌ 🔄❌          | ✅✅✅              | ✅✅✅ 📚           |
| 016-client-cookies             | ❌✅❌ 🔄❌          | ❌✅❌ 🔄❌          | ❌✅❌ 🔄❌ 📚       |
| 017-use-search-params          | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 018-use-router                 | ✅✅✅              | ✅✅✅              | ✅✅✅ ⚠️           |
| 019-use-action-state           | ✅✅✅ 🔄✅          | ✅✅✅              | ✅✅✅ 📚           |
| 020-no-use-effect              | ✅✅❌ 🔄❌          | ✅✅✅              | ✅✅✅ ⚠️           |
| 021-avoid-fetch-in-effect      | ✅✅✅              | ✅✅✅              | ✅✅✅ ⚠️           |
| 022-prefer-server-actions      | ✅✅✅              | ✅✅✅ 🔄✅          | ✅✅✅ ⚠️           |
| 023-avoid-getserversideprops   | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 024-avoid-redundant-usestate   | ✅✅✅              | ✅✅✅              | ✅✅✅ ⚠️           |
| 025-prefer-next-link           | ✅✅✅              | ✅✅✅              | ✅✅✅ ⚠️           |
| 026-no-serial-await            | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 027-prefer-next-image          | ✅✅✅              | ✅✅✅              | ✅✅✅ ⚠️           |
| 028-prefer-next-font           | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 029-use-cache-directive        | ❌✅✅ 🔄❌          | ✅✅✅              | ❌✅✅ 🔄❌ 📚       |
| 030-app-router-migration-hard  | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 031-ai-sdk-migration-simple    | ❌✅✅ 🔄❌          | ❌✅✅ 🔄❌          | ❌✅✅ 🔄❌ 📥       |
| 032-ai-sdk-model-specification-string | ✅✅❌ 🔄❌          | ✅✅❌ 🔄❌          | ✅✅❌ 🔄❌ 📚       |
| 033-ai-sdk-v4-model-specification-function | ❌✅❌ 🔄❌          | ❌✅✅ 🔄✅          | ❌✅❌ 🔄❌ 📚       |
| 034-ai-sdk-render-visual-info  | ❌✅❌ 🔄❌          | ❌✅❌ 🔄❌          | ❌✅❌ 🔄❌ 📚       |
| 035-ai-sdk-call-tools          | ❌✅❌ 🔄❌          | ❌✅❌ 🔄❌          | ❌✅❌ 🔄❌ 📚       |
| 036-ai-sdk-call-tools-multiple-steps | ❌✅✅ 🔄❌          | ❌✅✅ 🔄❌          | ❌✅✅ 🔄❌ ⚠️       |
| 037-ai-sdk-embed-text          | ❌✅❌ 🔄❌          | ❌✅❌ 🔄❌          | ❌✅❌ 🔄❌ 📚       |
| 038-ai-sdk-mcp                 | ❌✅✅ 🔄❌          | ❌✅✅ 🔄❌          | ❌✅❌ 🔄❌ 📚       |
| 039-parallel-routes            | ✅✅❌ 🔄❌          | ✅✅✅ 🔄✅          | ✅✅❌ 🔄❌ 📚       |
| 040-intercepting-routes        | ✅✅❌ 🔄❌          | ✅✅❌ 🔄❌          | ✅✅❌ 🔄❌ ⚠️       |
| 041-route-groups               | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 042-loading-ui                 | ✅✅✅              | ✅✅✅              | ✅✅✅ ⚠️           |
| 043-error-boundaries           | ❌✅✅ 🔄❌          | ❌✅✅ 🔄❌          | ❌✅✅ 🔄❌ 📚       |
| 044-metadata-api               | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 045-server-actions-form        | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 046-streaming                  | ✅✅✅              | ✅✅✅              | ✅✅✅ 🔄✅ ⚠️       |
| 047-middleware                 | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 048-draft-mode                 | ✅✅✅              | ✅✅✅              | ✅✅✅ 📚           |
| 049-revalidation               | ✅✅✅              | ✅✅❌ 🔄❌          | ✅✅✅ 🔄✅ ⚠️       |
```

## Recommendations

1. **Investigate CLAUDE.md regressions** (014-server-routing, 049-revalidation) - the docs might be giving conflicting or outdated guidance

2. **Debug SKILL.md trigger rate** - why isn't it activating for the ⚠️ cases? ~30% of evals don't trigger the skill

3. **Focus on hard failures** - parallel routes, intercepting routes, pathname-server, client-cookies need targeted documentation or examples

4. **AI SDK coverage needs work** - may need dedicated training data or specialized instructions for evals 031-038

5. **Consider doc quality** - pulling docs (📚) doesn't guarantee improvement; the content itself may need refinement

## Related

- nextjs-idiomatic-docs.md
- debugging-docs-bias-in-eval-regressions.md
- documentation-examples-bias-agent-behavior.md

## Date

2025-01-16
