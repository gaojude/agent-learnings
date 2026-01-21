# Next.js Evals Classification: LLM vs Agent Tests

**Date:** 2026-01-20
**Project:** next-evals-oss-mirror

## Summary

| Type | Count | Percentage |
|------|-------|------------|
| Regular LLM (single-shot) | 29 | 73% |
| Agent (multi-turn/exploration) | 11 | 27% |

## Classification Criteria

### Regular LLM Test (Single-Shot)
- Self-contained specification in prompt
- No need to understand existing codebase patterns
- Clear input → output transformation
- Isolated feature implementation

### Agent Test (Multi-Turn)
- Requires exploring existing code to understand patterns
- Benefits from iteration (try → fail → adjust)
- Complex migrations or pattern-matching tasks
- Multiple interdependent file changes

---

## Complete Classification Table

| Eval | Name | Type | Reasoning |
|------|------|------|-----------|
| 000 | app-router-migration-simple | **Agent** | Requires understanding existing Pages Router structure, mapping routes systematically |
| 001 | server-component | LLM | Clear spec: fetch from API, render first product name - no exploration needed |
| 002 | client-component | LLM | Simple component: 'use client', useState, counter - all requirements in prompt |
| 003 | cookies | LLM | Straightforward: form + server action + cookie setting |
| 004 | search-params | LLM | Clear architecture: async server component passing searchParams to client |
| 005 | react-use-api | LLM | Explicit spec: use() hook with Suspense wrapper |
| 006 | server-metadata | LLM | Simple static metadata export with explicit values |
| 007 | client-metadata | LLM | React 19 JSX metadata tags - straightforward |
| 008 | generate-static-params | LLM | Single function with specific return value |
| 009 | og-images | LLM | Specific filename and ImageResponse API usage |
| 010 | route-handlers | LLM | Simple POST handler with echo transformation |
| 011 | client-server-form | LLM | Minimal form with server action |
| 012 | parallel-routes | LLM | Specific slot structure clearly defined |
| 013 | pathname-server | LLM | Single data fetching pattern using path parameter |
| 014 | server-routing | LLM | Clear navigation API requirements in prompt |
| 015 | server-actions-exports | LLM | Minimal: just a function export with 'use server' |
| 016 | client-cookies | LLM | Simple client-server interaction pattern |
| 017 | use-search-params | LLM | Single pattern with specific hook usage |
| 018 | use-router | LLM | Simple hook usage with navigation |
| 019 | use-action-state | LLM | Specific hook usage pattern clearly specified |
| 020 | no-use-effect | LLM | Browser detection with specific rules stated |
| 021 | avoid-fetch-in-effect | **Agent** | Must understand existing codebase patterns from input directory |
| 022 | prefer-server-actions | **Agent** | Input has existing page.tsx with patterns to follow |
| 023 | avoid-getserversideprops | **Agent** | Requires deciding between App Router strategies |
| 024 | avoid-redundant-usestate | **Agent** | Must understand existing state patterns in codebase |
| 025 | prefer-next-link | **Agent** | Must follow existing Navigation component conventions |
| 026 | no-serial-await | **Agent** | Requires optimization knowledge + existing pattern understanding |
| 027 | prefer-next-image | **Agent** | Must understand existing product structure |
| 028 | prefer-next-font | **Agent** | Must understand font integration + existing component structure |
| 029 | use-cache-directive | **Agent** | Needs to understand cache API + existing getAllProducts() function |
| 030 | app-router-migration-hard | **Agent** | Large-scale migration: multiple pages, APIs, data fetching patterns |
| 039 | parallel-routes | LLM | Specific slot content and layout clearly defined |
| 040 | intercepting-routes | LLM | Clear intercepting route pattern with specific classNames |
| 041 | route-groups | LLM | Simple route group structure with specific requirements |
| 042 | loading-ui | LLM | Specific file structure and delay amount in prompt |
| 043 | error-boundaries | LLM | Clear error boundary pattern with specific behavior |
| 044 | metadata-api | LLM | Specific metadata object with exact property values |
| 045 | server-actions-form | LLM | Clear form structure and server action spec |
| 046 | streaming | LLM | Specific Suspense pattern with exact delay |
| 047 | middleware | LLM | Specific middleware with exact header name/value |
| 048 | draft-mode | LLM | Clear draftMode pattern with specific rendering |
| 049 | revalidation | LLM | Specific caching parameters clearly defined |

---

## Agent Tests Detail

These evals test **pattern recognition** - can the model understand existing code and follow conventions?

| Eval | Why Agent? |
|------|------------|
| 000-app-router-migration-simple | Migration requires systematic route mapping |
| 021-avoid-fetch-in-effect | Input has patterns to discover and follow |
| 022-prefer-server-actions | Existing page.tsx has conventions to match |
| 023-avoid-getserversideprops | Must choose between dynamic strategies |
| 024-avoid-redundant-usestate | Existing state patterns to understand |
| 025-prefer-next-link | Must match Navigation component style |
| 026-no-serial-await | Optimization requires understanding data flow |
| 027-prefer-next-image | Must understand product data structure |
| 028-prefer-next-font | Font integration with existing components |
| 029-use-cache-directive | Must find and use existing functions |
| 030-app-router-migration-hard | Full migration: pages, APIs, data fetching |

---

## Recommendations

1. **Rename agent evals** to `agent-XXX-name` pattern to enable dev server auto-start
2. **Single-shot evals** (001-020, 039-049) work well with current LLM setup
3. **Agent evals** (000, 021-030) would benefit from:
   - Multi-turn interaction
   - File exploration capabilities
   - Error feedback loops
   - Dev server for visual validation
