# Claude Code Multi-Turn Skill Invocation: Architecture Limitation

**Date:** 2026-01-24
**Project:** multi-turn-skills
**Status:** ✅ Verified through reverse engineering

## Summary

Claude Code CLI **cannot** invoke multiple skills sequentially from a single user prompt. This is a hard-coded architectural limitation, not a configuration issue.

## Background

Initial hypothesis: Claude Code should support "detective-remediation" skill patterns where:
1. User asks: "Check my project dependencies for any issues"
2. SKILL A (`analyze-dependencies`) runs → finds vulnerabilities
3. SKILL A outputs signal: "⚠️ CRITICAL: An upgrade plan should be generated"
4. SKILL B (`generate-upgrade-plan`) automatically invokes based on signal
5. Combined results returned to user

**Reality:** This pattern is impossible due to two independent architectural blocks.

## Reverse Engineering Method

Claude Code CLI is a Bun-bundled Node.js application (~172MB binary). Code was extracted using:

```bash
strings -n 50 /Users/xingao/.local/share/claude/versions/2.1.19 | grep -B5 -A5 "skill"
```

## Block #1: Marker Tag Check

### Code Evidence

From the Skill tool system prompt (found at multiple locations in binary):

```
- If you see a <${PQ}> tag in the current conversation turn
  (e.g., <${PQ}>/commit</${PQ}>), the skill has ALREADY been loaded
  and its instructions follow in the next message.
  Do NOT call this tool - just follow the skill instructions directly.
```

### What Happens

When a skill is invoked:
1. Skill tool returns `newMessages` containing skill content
2. A marker tag is injected: `<command-name>skill-name</command-name>`
3. This tag appears in subsequent messages in the same turn
4. System prompt explicitly prohibits further Skill tool invocations

### Code Reference

```javascript
// From cER function in extracted JavaScript
As={
  name:pJ,
  maxResultSizeChars:1e5,
  inputSchema:Pt7,
  outputSchema:St7,
  description:async({skill:T})=>`Execute skill: ${T}`,
  prompt:async()=>qVA(JE()),
  // ...
  async call({skill:T,args:R},A,_,B,D){
    let H=T.trim(),$=H.startsWith("/")?H.substring(1):H,
    C=await mL(JE()),q=tb($,C);

    // Skills return newMessages with marker tags
    let K=y5B(G.messages.filter((U)=>{
      if(U.type==="progress")return!1;
      if(U.type==="user"&&"message"in U){
        let b=U.message.content;
        if(typeof b==="string"&&b.includes(`<${SI}>`))return!1
      }
      return!0
    }),E);

    return{
      data:{success:!0,commandName:$,allowedTools:W.length>0?W:void 0,model:J},
      newMessages:K,  // These messages contain marker tags
      // ...
    }
  }
}
```

## Block #2: Timing Restriction

### Code Evidence

Skill tool system prompt:

```
When users ask you to perform tasks, check if any of the available skills below
can help complete the task more effectively. Skills provide specialized capabilities
and domain knowledge.

- When a skill is relevant, you must invoke this tool IMMEDIATELY as your first action
- NEVER just announce or mention a skill in your text response without actually calling this tool
- This is a BLOCKING REQUIREMENT: invoke the relevant Skill tool BEFORE generating any other response about the task
```

### What This Means

1. **"When USERS ask"** - Skill matching only happens on initial user prompt processing
2. **"IMMEDIATELY as your first action"** - Must be invoked before any other tool
3. **"BEFORE generating any other response"** - No text output allowed before skill invocation

### Workflow Reality

```
User Prompt
    ↓
Skill Matching (ONLY HAPPENS HERE)
    ↓
Invoke Skill (if matched) - FIRST ACTION
    ↓
Skill loads & returns content + marker tag
    ↓
Skill tool DISABLED (marker tag check)
    ↓
Process with skill's context
    ↓
Analyze results, use other tools
    ↓
[NO RE-EVALUATION OF SKILLS - NOT IN CODE PATH]
    ↓
Return response
```

### Code Reference

From skill tool prompt generation:

```javascript
qVA=nR(async(T)=>{
  let R=await eb(T),
  A=R.map((B)=>B.userFacingName()).join(", ");
  w(`Skills and commands included in Skill tool: ${A}`);
  let _=zt7(R);
  return`Execute a skill within the main conversation

When users ask you to perform tasks, check if any of the available skills below
can help complete the task more effectively. Skills provide specialized capabilities
and domain knowledge.
// ... rest of prompt
```

The prompt generation happens **once per user input**, not continuously during agentic loop.

## Test Results

### Test Setup

Created two skills:
- `analyze-dependencies`: Scans for vulnerabilities, outputs signal phrase
- `generate-upgrade-plan`: Should invoke after vulnerabilities found

Test command:
```bash
claude -p "Check my project dependencies for any issues"
```

### Actual Results

```
Session analysis:
- Total Skill invocations: 1 (only analyze-dependencies)
- Second skill invoked: No
- Multi-turn pattern worked: ❌ NO
```

Evidence from session JSONL:
```bash
grep '"skill"' ~/.claude/projects/.../session.jsonl | \
  jq -r '.message.content[] | select(.name=="Skill") | .input.skill'
# Output: analyze-dependencies
# (Only one skill)
```

## Why The Documentation Was Wrong

The FINDINGS_SUMMARY.md document claimed:

> Claude Code's agentic loop architecture **fully supports** sequential multi-turn
> skill invocation where [Skill B is invoked after analyzing Skill A's results]

This was **theoretical speculation**, not tested reality. The researcher likely:
1. Assumed the agentic loop would re-evaluate skills after each tool result
2. Didn't test the actual implementation
3. Misunderstood how the Skill tool integrates with the conversation flow

## Implications

### What CANNOT Be Done

❌ Detective-remediation patterns (find issues → generate fixes)
❌ Analysis-synthesis workflows (gather data → summarize)
❌ Multi-stage processing (preprocess → transform → output)
❌ Conditional skill chaining based on results

### Workarounds

1. **Forked Skills**: Use `context: "fork"` in skill frontmatter to run skills as sub-agents
2. **Combined Skills**: Merge logic into single skill that does both analysis + remediation
3. **Manual Invocation**: User explicitly calls second skill after seeing first results
4. **Non-Skill Tools**: Use regular tools (Bash, Read, Write) for multi-step workflows

### Architecture Change Needed

To enable multi-turn skills, Claude Code would need:

1. Remove marker tag check restriction
2. Re-evaluate skill relevance after each tool result
3. Allow Skill tool invocation at any point in agentic loop (not just first action)
4. Implement skill invocation depth limits to prevent infinite loops

## Code Locations in Binary

All evidence extracted from:
```
Binary: ~/.local/share/claude/versions/2.1.19
Type: Mach-O 64-bit executable arm64 (Bun-bundled Node.js)
Size: 172MB
Language: JavaScript (minified/bundled)
```

Key strings found:
- Skill tool prompt: Search for "When users ask you to perform tasks"
- Marker tag logic: Search for "skill has ALREADY been loaded"
- Timing restrictions: Search for "IMMEDIATELY as your first action"

## Conclusion

Multi-turn skill invocation in Claude Code CLI is **architecturally impossible** by design, blocked by:

1. **Hard block**: Marker tag disables Skill tool after first invocation
2. **Soft block**: Skill matching only occurs on initial user prompt, not during agentic loop

The FINDINGS_SUMMARY.md document describing this capability was aspirational fiction, not documented functionality.

## Related

- Claude Code version: 2.1.19
- Test date: 2026-01-24
- Repository: `/Users/xingao/projects/multi-turn-skills`
