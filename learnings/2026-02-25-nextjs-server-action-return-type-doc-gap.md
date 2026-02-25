# Next.js Docs Gap: Server Action Return Type Constraint for Form Actions

**Date:** 2026-02-25
**Project:** next-evals-oss / nextdocfs

## The Gap

The Next.js docs at `app/getting-started/updating-data` show the correct pattern for server actions used as form actions, but **never explicitly state that form actions must return `void`**.

The example shows:
```ts
'use server'
import { refresh } from 'next/cache'

export async function updatePost(formData: FormData) {
  // Update data
  refresh()
}
```

This implicitly returns void, but nowhere do the docs say:
- Server actions passed to `<form action={...}>` **must** return `void | Promise<void>`
- Returning a value (e.g. `return newValue`) will cause a TypeScript error

## Evidence

Found during eval testing with `nextdocfs` CLI (agent-038-refresh-settings):

- **Sonnet 4.5 run-3** found the `refresh()` API via the CLI, used it correctly (`import { refresh } from 'next/cache'`), passed all 6 eval assertions — but **failed the build** because the server action returned `"on" | "off"` instead of void
- The type error: `Type '() => Promise<"off" | "on">` is not assignable to type `(formData: FormData) => void | Promise<void>`
- Sonnet 4.6 didn't hit this issue — it followed the example pattern implicitly

## Impact

- Weaker models that understand the API but don't infer the void constraint from examples will fail
- The docs page `app/getting-started/updating-data` has 16 mentions of `return` but zero mentions of `void` or the return type constraint
- A single sentence like "Server actions used as form actions must not return a value" would fix this

## Where to Fix

- `https://nextjs.org/docs/app/getting-started/updating-data` — in the Forms section or the Refreshing section
- Possibly also in `https://nextjs.org/docs/app/api-reference/functions/refresh`

## Search Commands Used

```bash
nextdocfs grep "refresh" app/getting-started/updating-data
nextdocfs grep "return" app/getting-started/updating-data
nextdocfs grep "void" app/getting-started/updating-data  # no matches
```
