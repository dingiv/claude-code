# 工具执行

## 总览

工具执行是一个三阶段管道：**发现 → 权限检查 → 执行**。每个工具是一个实现了 `Tool<I,O,P>` 接口的对象，通过 `buildTool()` 工厂创建。

```
tool_use block（来自 API 响应）
    ↓
runToolUse()                          ← 统一入口
    ├─ findToolByName()               ← 1. 工具发现
    ├─ inputSchema.safeParse()        ← 2. 输入验证
    ├─ checkPermissionsAndCallTool()  ← 3. 权限检查
    └─ tool.call()                    ← 4. 执行
    ↓
tool_result UserMessage（返回给 API）
```

## 核心文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/Tool.ts` | — | Tool 接口定义、buildTool 工厂、工具查找 |
| `src/services/tools/toolExecution.ts` | — | runToolUse() — 统一执行入口 |
| `src/services/tools/StreamingToolExecutor.ts` | 560 | 流式并行执行器 |
| `src/services/tools/toolOrchestration.ts` | 207 | 非流式串行/并行编排 |
| `src/utils/permissions/permissions.ts` | — | 权限检查管道 |
| `packages/builtin-tools/src/tools/` | 60 目录 | 59+ 工具实现 |

## Tool 接口

```typescript
type Tool<Input, Output, Progress> = {
  name: string                         // 工具名（API 层标识）
  aliases?: string[]                   // 废弃名兼容
  inputSchema: ZodType<Input>          // Zod 输入 schema
  description(input): Promise<string>  // 动态描述（根据输入生成）
  call(input, context): Promise<ToolResult<Output>>  // 执行逻辑
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>
  isConcurrencySafe(input): boolean    // 是否可并行执行
  isEnabled(): boolean                 // 是否启用
  isReadOnly(input): boolean           // 是否只读
  isDestructive?(input): boolean       // 是否有破坏性
  interruptBehavior?(): 'cancel' | 'block'  // 中断行为
}
```

### buildTool() 工厂默认值

```typescript
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: () => false,    // 默认不并发安全
  isReadOnly: () => false,
  isDestructive: () => false,
  checkPermissions: () => Promise.resolve({ behavior: 'allow' }),
}
```

## runToolUse() 执行流程

### 阶段 1：工具发现

```typescript
// toolExecution.ts:372-440
let tool = findToolByName(toolUseContext.options.tools, toolName)

// 兼容废弃工具别名
if (!tool) {
  const fallbackTool = findToolByName(getAllBaseTools(), toolName)
  if (fallbackTool?.aliases?.includes(toolName)) {
    tool = fallbackTool
  }
}

// 找不到 → 返回错误 tool_result
if (!tool) {
  yield { message: createUserMessage({
    content: [{ type: 'tool_result', content: `Error: No such tool: ${toolName}`, is_error: true }]
  })}
  return
}
```

### 阶段 2：输入验证

```typescript
// Zod schema 解析
const parsedInput = tool.inputSchema.safeParse(toolUse.input)

// 可选的额外验证
if (tool.validateInput) {
  const validation = await tool.validateInput(parsedInput.data, toolUseContext)
  if (!validation.valid) {
    yield { message: createUserMessage({
      content: [{ type: 'tool_result', content: validation.error, is_error: true }]
    })}
    return
  }
}
```

### 阶段 3：权限检查

权限系统是一个多层管道：

```
checkPermissionsAndCallTool()
    │
    ├─ PreToolUse hooks（前置钩子，可拦截）
    │
    ├─ hasPermissionsToUseTool()（规则匹配）
    │   ├─ allow 规则匹配 → behavior: 'allow'
    │   ├─ deny 规则匹配 → behavior: 'deny'
    │   ├─ ask 规则匹配 → behavior: 'ask'
    │   ├─ auto 模式 → classifierDecision（AI 安全分类器）
    │   └─ 无匹配 → 根据 mode 决定（default/dontAsk/auto）
    │
    ├─ behavior === 'allow' → 直接执行
    ├─ behavior === 'deny' → 返回拒绝消息
    └─ behavior === 'ask' → 等待用户在 REPL 中批准/拒绝
```

### 阶段 4：执行

```typescript
// toolExecution.ts — 简化
const result = await tool.call(parsedInput.data, toolUseContext)

// 结果转为 tool_result block
const toolResultBlock = {
  type: 'tool_result',
  tool_use_id: toolUse.id,
  content: result.output,  // 文本或结构化内容
  is_error: result.isError,
}

yield {
  message: createUserMessage({
    content: [toolResultBlock],
    toolUseResult: result.output,
    sourceToolAssistantUUID: assistantMessage.uuid,
  })
}
```

## 权限系统详解

### 权限模式

| 模式 | 说明 |
|------|------|
| `default` | 标准交互模式，逐次询问 |
| `auto` | 自动批准，通过 AI 分类器判断 |
| `dontAsk` | 全部自动批准 |
| `acceptEdits` | 自动批准文件编辑，其他询问 |
| `bypassPermissions` | 跳过所有权限检查（sandbox/root 环境） |
| `plan` | 规划模式，受限权限 |

### 规则来源（优先级从高到低）

```typescript
PERMISSION_RULE_SOURCES = [
  'cliArg',           // 命令行参数
  'command',          // 当前命令上下文
  'session',          // 会话级规则
  'userSettings',     // ~/.claude/settings.json
  'projectSettings',  // .claude/settings.json
  'localSettings',    // .claude/settings.local.json
  'flagSettings',     // feature flag 设置
  'policySettings',   // 管理员策略
]
```

### 规则匹配

规则支持三种匹配方式：

```typescript
// 1. 工具名精确匹配
"BashTool" → 匹配 BashTool

// 2. 命令模式（glob）
"bash(npm test)" → 匹配 npm test 命令
"Read(*.ts)" → 匹配所有 .ts 文件读取

// 3. 路径前缀
"Edit(src/)" → 匹配 src/ 目录下的编辑
```

### Auto 模式：AI 安全分类器

```typescript
// yoloClassifier.ts
if (permissionMode === 'auto') {
  const classification = classifyYoloAction(toolName, input)
  // → 'safe' | 'unsafe' | 'unknown'
  // safe → allow
  // unsafe → ask user
  // unknown → ask user
}
```

分类器是轻量级模型调用（Haiku），分析命令意图判断安全性。

## 工具结果格式化

### 大输出处理

超过 30K 字符的工具输出会被持久化到磁盘：

```typescript
mapToolResultToToolResultBlockParam(output, toolUseId) {
  if (output.length > 30000) {
    // 写入磁盘文件
    const filepath = persistToolOutput(output)
    return {
      type: 'tool_result',
      content: buildLargeToolResultMessage({
        filepath,          // 磁盘路径
        preview: output.slice(0, 30000),  // 预览
        hasMore: true,     // 提示有更多内容
      }),
    }
  }
  // 正常输出
  return { type: 'tool_result', content: output }
}
```

### 进度消息

长时间运行的工具可以 yield 进度消息（实时显示在 UI 中）：

```typescript
// tool.call() 内部
onProgress?.({ type: 'progress', message: 'Reading file...', percent: 50 })
```

进度消息通过 `StreamingToolExecutor` 的 `pendingProgress` 队列立即 yield 出去，不等工具完成。

## StreamingToolExecutor 详解

### 状态机

每个工具经历四个状态：

```
queued → executing → completed → yielded
```

### 并发控制规则

```typescript
canExecuteTool(isConcurrencySafe: boolean): boolean {
  const executing = tools.filter(t => t.status === 'executing')
  return (
    executing.length === 0 ||                                    // 无工具在执行
    (isConcurrencySafe && executing.every(t => t.isConcurrencySafe))  // 都是并发安全
  )
}
```

**示例**：模型同时调用 Read(a.ts) + Read(b.ts) + Bash(make)

```
时间线:
t0: Read(a) added → 立即执行（无工具在运行）
t1: Read(b) added → 立即执行（Read 并发安全，且正在执行的工具也并发安全）
t2: Bash(make) added → 排队（Bash 不并发安全，前面有工具在执行）
t3: Read(a) completed → processQueue() → Bash(make) 开始执行
t4: Read(b) completed
t5: Bash(make) completed
```

### Bash 错误级联

```typescript
// StreamingToolExecutor.ts:386-389
if (tool.block.name === BASH_TOOL_NAME && isErrorResult) {
  this.hasErrored = true
  this.siblingAbortController.abort('sibling_error')
}
```

Bash 错误会中止所有兄弟工具。原因是 Bash 命令通常有隐式依赖链（如 `mkdir` 失败后后续命令无意义）。

Read/WebFetch 等独立工具的错误不会级联。

### 中断处理

```
用户按 ESC
    ↓
abortController.abort()
    ↓
StreamingToolExecutor 检测到 signal.aborted
    ├─ 正在执行的工具 → 等待完成或生成 synthetic error
    ├─ 排队中的工具 → 生成 synthetic error
    └─ interruptBehavior === 'block' 的工具 → 不中断
```

## ToolUseContext

```typescript
type ToolUseContext = {
  options: {
    tools: Tools               // 所有可用工具
    mainLoopModel: string      // 当前模型
    thinkingConfig: ThinkingConfig
    mcpClients: MCPServerConnection[]
    agentDefinitions: { activeAgents, allAgents }
    isNonInteractiveSession: boolean
    // ...
  }
  abortController: AbortController  // 取消信号
  messages: Message[]                // 当前对话历史
  readFileState: FileStateCache      // 文件读取缓存
  getAppState(): AppState            // 获取应用状态
  setAppState(fn): void              // 更新应用状态
  queryTracking?: QueryChainTracking // 查询链追踪
  agentId?: string                   // 子 agent ID
  langfuseTrace?: LangfuseSpan       // 可观测性
}
```

ToolUseContext 是**不可变传递**的——每个工具调用收到自己的拷贝。Context modifier 机制允许工具修改上下文（如修改 permission rules），修改会通过 `contextModifier` 传递回来。

## 工具执行完整数据流

```
API 响应: tool_use block { name: "BashTool", input: { command: "npm test" } }
    ↓
query.ts: streamingToolExecutor.addTool(block, assistantMessage)
    ├─ findToolByName("BashTool") → BashTool 对象
    ├─ BashTool.isConcurrencySafe(input) → false
    └─ processQueue() → canExecuteTool? → 执行
    ↓
StreamingToolExecutor.executeTool(tool)
    ├─ runToolUse(block, assistantMessage, canUseTool, context)
    │   ├─ findToolByName → BashTool
    │   ├─ inputSchema.safeParse(input) → { command: "npm test" }
    │   ├─ checkPermissions(input, context)
    │   │   ├─ hasPermissionsToUseTool() → 检查 allow/deny/ask 规则
    │   │   └─ behavior === 'allow' → 继续
    │   ├─ BashTool.call({ command: "npm test" }, context)
    │   │   ├─ spawn subprocess
    │   │   ├─ capture stdout/stderr
    │   │   ├─ yield progress messages
    │   │   └─ return { output: "Tests: 42 passed", isError: false }
    │   └─ yield { message: createUserMessage({
    │         content: [{ type: 'tool_result', tool_use_id, content: "Tests: 42 passed" }]
    │       })}
    └─ tool.status = 'completed'
    ↓
query.ts: getCompletedResults() / getRemainingResults()
    ├─ yield tool_result message → REPL 显示
    └─ toolResults.push(message) → 加入对话历史
    ↓
下一轮 query() 调用时，tool_result 作为 UserMessage 发送给 API
```
