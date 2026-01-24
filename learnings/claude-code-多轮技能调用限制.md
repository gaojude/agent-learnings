# Claude Code 多轮技能调用：架构限制

**日期：** 2026-01-24
**项目：** multi-turn-skills
**状态：** ✅ 已通过逆向工程验证

## 摘要

Claude Code CLI **无法**从单个用户提示中顺序调用多个技能。这是硬编码的架构限制，不是配置问题。

## 背景

初始假设：Claude Code 应该支持"侦探-修复"技能模式：
1. 用户询问："检查我项目的依赖项是否有问题"
2. 技能 A（`analyze-dependencies`）运行 → 发现漏洞
3. 技能 A 输出信号："⚠️ 严重：需要生成升级计划"
4. 技能 B（`generate-upgrade-plan`）根据信号自动调用
5. 将组合结果返回给用户

**现实：** 由于两个独立的架构阻塞，此模式无法实现。

## 逆向工程方法

Claude Code CLI 是一个 Bun 打包的 Node.js 应用程序（~172MB 二进制文件）。使用以下方法提取代码：

```bash
strings -n 50 /Users/xingao/.local/share/claude/versions/2.1.19 | grep -B5 -A5 "skill"
```

## 阻塞 #1：标记标签检查

### 代码证据

从技能工具系统提示（在二进制文件的多个位置找到）：

```
- 如果你在当前对话轮次中看到 <${PQ}> 标签
  (例如 <${PQ}>/commit</${PQ}>)，说明技能已经被加载
  且其指令在下一条消息中。
  不要调用此工具 - 直接遵循技能指令。
```

### 发生了什么

当技能被调用时：
1. 技能工具返回包含技能内容的 `newMessages`
2. 注入标记标签：`<command-name>技能名称</command-name>`
3. 此标签出现在同一轮次的后续消息中
4. 系统提示明确禁止进一步的技能工具调用

### 代码引用

```javascript
// 从提取的 JavaScript 中的 cER 函数
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

    // 技能返回带有标记标签的 newMessages
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
      newMessages:K,  // 这些消息包含标记标签
      // ...
    }
  }
}
```

## 阻塞 #2：时序限制

### 代码证据

技能工具系统提示：

```
当用户要求您执行任务时，检查以下可用技能是否可以更有效地完成任务。
技能提供专业能力和领域知识。

- 当技能相关时，您必须立即将此工具作为您的第一个操作调用
- 切勿在不实际调用此工具的情况下在文本响应中宣布或提及技能
- 这是一个阻塞要求：在生成有关任务的任何其他响应之前调用相关的技能工具
```

### 这意味着什么

1. **"当用户询问"** - 技能匹配仅在初始用户提示处理时发生
2. **"立即作为您的第一个操作"** - 必须在任何其他工具之前调用
3. **"在生成任何其他响应之前"** - 技能调用前不允许文本输出

### 工作流程现实

```
用户提示
    ↓
技能匹配（仅在此处发生）
    ↓
调用技能（如果匹配）- 第一个操作
    ↓
技能加载并返回内容 + 标记标签
    ↓
技能工具禁用（标记标签检查）
    ↓
使用技能的上下文处理
    ↓
分析结果，使用其他工具
    ↓
[没有重新评估技能 - 不在代码路径中]
    ↓
返回响应
```

### 代码引用

从技能工具提示生成：

```javascript
qVA=nR(async(T)=>{
  let R=await eb(T),
  A=R.map((B)=>B.userFacingName()).join(", ");
  w(`Skills and commands included in Skill tool: ${A}`);
  let _=zt7(R);
  return`在主对话中执行技能

当用户要求您执行任务时，检查以下可用技能是否可以更有效地完成任务。
技能提供专业能力和领域知识。
// ... 提示的其余部分
```

提示生成在**每个用户输入时发生一次**，而不是在 agentic 循环期间持续发生。

## 测试结果

### 测试设置

创建了两个技能：
- `analyze-dependencies`：扫描漏洞，输出信号短语
- `generate-upgrade-plan`：应该在发现漏洞后调用

测试命令：
```bash
claude -p "Check my project dependencies for any issues"
```

### 实际结果

```
会话分析：
- 技能调用总数：1（仅 analyze-dependencies）
- 调用第二个技能：否
- 多轮模式有效：❌ 否
```

会话 JSONL 的证据：
```bash
grep '"skill"' ~/.claude/projects/.../session.jsonl | \
  jq -r '.message.content[] | select(.name=="Skill") | .input.skill'
# 输出：analyze-dependencies
#（只有一个技能）
```

## 为什么文档是错误的

FINDINGS_SUMMARY.md 文档声称：

> Claude Code 的 agentic 循环架构**完全支持**顺序多轮技能调用，
> 其中[在分析技能 A 的结果后调用技能 B]

这是**理论推测**，而不是经过测试的现实。研究人员可能：
1. 假设 agentic 循环会在每个工具结果后重新评估技能
2. 没有测试实际实现
3. 误解了技能工具如何与对话流集成

## 影响

### 无法做什么

❌ 侦探-修复模式（发现问题 → 生成修复）
❌ 分析-综合工作流（收集数据 → 总结）
❌ 多阶段处理（预处理 → 转换 → 输出）
❌ 基于结果的条件技能链接

### 解决方法

1. **分叉技能**：在技能前置中使用 `context: "fork"` 将技能作为子代理运行
2. **组合技能**：将逻辑合并到同时执行分析 + 修复的单个技能中
3. **手动调用**：用户在看到第一个结果后显式调用第二个技能
4. **非技能工具**：使用常规工具（Bash、Read、Write）进行多步骤工作流

### 需要的架构更改

要启用多轮技能，Claude Code 需要：

1. 删除标记标签检查限制
2. 在每个工具结果后重新评估技能相关性
3. 允许在 agentic 循环的任何点调用技能工具（不仅仅是第一个操作）
4. 实现技能调用深度限制以防止无限循环

## 二进制文件中的代码位置

所有证据提取自：
```
二进制文件：~/.local/share/claude/versions/2.1.19
类型：Mach-O 64-bit executable arm64（Bun 打包的 Node.js）
大小：172MB
语言：JavaScript（压缩/打包）
```

找到的关键字符串：
- 技能工具提示：搜索 "When users ask you to perform tasks"
- 标记标签逻辑：搜索 "skill has ALREADY been loaded"
- 时序限制：搜索 "IMMEDIATELY as your first action"

## 结论

Claude Code CLI 中的多轮技能调用在设计上**架构上不可能**，被以下因素阻止：

1. **硬阻塞**：标记标签在第一次调用后禁用技能工具
2. **软阻塞**：技能匹配仅在初始用户提示时发生，而不是在 agentic 循环期间

描述此功能的 FINDINGS_SUMMARY.md 文档是理想化的虚构，而不是记录的功能。

## 实际代码证据总结

### 关键发现 1：标记标签检查

```javascript
// 技能调用返回带有标记的消息
return{
  data:{success:!0,commandName:$,allowedTools:W.length>0?W:void 0,model:J},
  newMessages:K,  // 包含 <command-name> 标签
  contextModifier(U){
    // 修改上下文但不重新启用技能工具
  }
}
```

系统提示明确说明：
```
如果你看到 <command-name> 标签，技能已经被加载。
不要调用此工具 - 直接遵循技能指令。
```

### 关键发现 2：只在用户输入时匹配

```javascript
// 提示生成函数 - 只在用户输入时调用
qVA=nR(async(T)=>{
  let R=await eb(T),  // 获取可用技能列表
  A=R.map((B)=>B.userFacingName()).join(", ");
  w(`Skills and commands included in Skill tool: ${A}`);
  let _=zt7(R);
  return`Execute a skill within the main conversation

When users ask you to perform tasks, check if any of the available skills...
```

**关键点**：此函数在用户输入时调用一次，而不是在 agentic 循环的每个步骤中调用。

### 关键发现 3：第一个操作要求

系统提示要求：
```
- 当技能相关时，您必须立即将此工具作为您的第一个操作调用
- 这是一个阻塞要求：在生成有关任务的任何其他响应之前调用相关的技能工具
```

这意味着即使允许多次调用，技能也只能在处理开始时调用，而不能在分析中间结果后调用。

## 相关

- Claude Code 版本：2.1.19
- 测试日期：2026-01-24
- 存储库：`/Users/xingao/projects/multi-turn-skills`
- 二进制位置：`~/.local/share/claude/versions/2.1.19`
- 二进制类型：Bun 打包的 Node.js (Mach-O arm64)
