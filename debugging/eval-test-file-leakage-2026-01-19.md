# Eval Test File Leakage Bug - Baseline Can Reverse-Engineer Requirements

## Summary

Critical eval design flaw: test files are accessible in input folders, allowing baseline Claude Code to "cheat" by reading tests and reverse-engineering exact requirements, giving it an unfair advantage over documentation-guided approaches.

## Context

Discovered while investigating why `022-prefer-server-actions` eval showed baseline Claude Code passing while `+Docs` and `+Skill2` variants failed. The failure was due to missing form validation logic.

## Details

### The Problem

All eval input folders include test files (e.g., `page.test.tsx`) with NO restrictions on reading them:

```
evals/022-prefer-server-actions/input/app/
├── ContactForm.tsx
├── layout.tsx
├── page.test.tsx    ← Test file is accessible!
└── page.tsx
```

### Evidence from Transcripts

**Baseline Claude Code** read the test file and extracted exact requirements:

```
Files read by baseline:
- ContactForm.tsx
- page.test.tsx  ← READ THE TEST FILE!

Model's response after reading test:
"Now I have a clear understanding of the requirements...
Has form validation checking `!name || !email || !message`"
```

The test file contains the exact regex pattern expected:
```typescript
test('includes form validation', () => {
  expect(content).toMatch(/!name\s*\|\|\s*!email\s*\|\|\s*!message|if\s*\([^)]*(!name|!email|!message)/);
});
```

**+Docs and +Skill2** followed documentation patterns instead:
- +Docs: Read only source files, copied existing pattern without validation
- +Skill2: Read `08-updating-data.mdx` docs which have NO validation examples

### Impact on Results

| Variant | Files Read | Result |
|---------|-----------|--------|
| Baseline CC | `page.test.tsx` + source | ✅ Passed (reverse-engineered from test) |
| +Docs | Source files only | ❌ Failed (no validation) |
| +Skill2 | Docs + source files | ❌ Failed (docs lack validation) |

### Affected Evals

ALL evals have this issue - test files in input folders:
```
evals/000-app-router-migration-simple/input/migration.test.tsx
evals/001-server-component/input/app/page.test.tsx
evals/002-client-component/input/app/page.test.tsx
... (20+ more evals)
```

### Root Cause

1. Test files included in `input/` folders from initial commit
2. No `.gitignore` or `.claudeignore` to exclude `*.test.*` files
3. Prompt only says "Do not RUN tests" not "Do not READ test files"

```typescript
// Current prompt restriction (lib/claude-code-runner.ts)
const enhancedPrompt = `${prompt}
IMPORTANT: Do not run npm, pnpm, yarn, or any package manager commands.
Dependencies have already been installed. Do not run build, test, or dev
server commands. Just write the code files.`;
// ← Says nothing about not READING test files!
```

### NOT a Regression

Git history confirms this was never restricted:
- Initial commit (eaf7596): Test files always in input folders
- Commit 742fde9: Added "do not RUN tests" but never "do not READ tests"

## Fix Options

1. **Add to each input's .gitignore:**
   ```
   *.test.*
   *.spec.*
   ```

2. **Move test files outside input folder:**
   ```
   evals/022-prefer-server-actions/
   ├── input/           ← Code only
   ├── tests/           ← Test files here
   └── prompt.md
   ```

3. **Add prompt restriction:**
   ```
   Do not read any test files (*.test.*, *.spec.*).
   Implement based on requirements, not test expectations.
   ```

4. **Use .claudeignore** (if supported):
   ```
   *.test.tsx
   *.test.ts
   *.spec.tsx
   *.spec.ts
   ```

## Related

- Repo: `next-evals-oss`
- Eval: `022-prefer-server-actions`
- File: `lib/claude-code-runner.ts`
- Tags: #eval #testing #bias #next-evals
