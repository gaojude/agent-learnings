# Skill Detection via Transcript Verification

## Summary
Verify Claude skill usage by parsing conversation transcripts (.jsonl) for skill invocation patterns and doc file reads.

## Context
When running evals that compare different approaches (baseline vs skill-enhanced), we need to verify that Claude actually used the skill and read the documentation. This is important for accurate eval metrics.

## Details

### Transcript Location
Claude stores conversation transcripts in:
```
~/.claude/projects/<encoded-project-path>/*.jsonl
```

Where `<encoded-project-path>` is the project directory with `/` replaced by `-`.

### Detection Approach

#### 1. Skill Invocation Detection
Check if the Skill tool was called with the skill name:

```typescript
// Look for these patterns in transcript
content.includes('"skill":"nextjs-doc"') ||
content.includes('"skill": "nextjs-doc"') ||
content.includes('Skill (/nextjs-doc)')
```

#### 2. Docs Read Detection
Check if documentation files (.mdx) were read from expected paths:

```typescript
// Bundled docs (eager loading - v0.0.8+)
const bundledMdxMatches = content.match(/[^"]*\.claude\/skills\/[^"]*\.mdx/g);

// Lazy-loaded docs (older versions)
const lazyMdxMatches = content.match(/\/(?:tmp|var\/folders)[^"]*next-skills[^"]*\.mdx/g);
```

### Verification Status Indicators
| Indicator | Meaning |
|-----------|---------|
| 📚 | Skill invoked AND docs read |
| 📥 | Skill invoked but docs NOT read |
| ⚠️ | Skill NOT invoked |

### Example Implementation

```typescript
export async function verifySkillUsage(outputDir: string): Promise<{
  skillUsed: boolean;
  skillInvoked: boolean;
  docsRead: boolean;
  docsFilesRead: string[];
}> {
  // 1. Find transcript file
  const projectPathEncoded = outputDir.replace(/\//g, '-');
  const claudeProjectsDir = path.join(homedir(), '.claude', 'projects', projectPathEncoded);

  // 2. Read transcript content
  const content = await fs.readFile(transcriptPath, 'utf-8');

  // 3. Check for skill invocation
  if (content.includes('"skill":"nextjs-doc"')) {
    result.skillInvoked = true;
  }

  // 4. Check for docs being read
  const mdxMatches = content.match(/\.claude\/skills\/[^"]*\.mdx/g);
  if (mdxMatches?.length > 0) {
    result.docsRead = true;
  }

  return result;
}
```

### Testing the Patterns
Always verify regex patterns match actual transcript content:

```bash
# Check skill invocation
grep -o '"skill":"nextjs-doc"' transcript.jsonl

# Check docs read
grep -o '\.claude/skills/[^"]*\.mdx' transcript.jsonl
```

## Related
- next-evals-oss project
- @judegao/next-skills package
- Claude Code skills system
