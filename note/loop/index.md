# Agent 循环主线：从用户输入到工具执行

## 总览

整个循环只有两个函数：`query()` 入口 + `queryLoop()` 无限循环。`queryLoop` 的每次迭代就是一次"模型调用 → 工具执行 → 判断是否继续"的完整轮次。

```
query() (267-353行)
  └── yield* queryLoop() (355-1945行)
        while(true) {
          ① 上下文预处理
          ② 调用 API（流式）
          ③ 处理流式响应
          ④ 执行工具
          ⑤ 判断是否继续 → state = next; continue
        }
```

## 核心文件

[](/src/query.ts)

| 文件                                          | 行数 | 职责                                      |
| --------------------------------------------- | ---- | ----------------------------------------- |
| `src/query.ts`                                | 1945 | Agent 循环主体（query + queryLoop）       |
| `src/services/api/claude.ts`                  | 3570 | 流式 API 通信（请求构建 + SSE 处理）      |
| `src/context.ts`                              | 189  | 上下文收集（git status、CLAUDE.md、日期） |
| `src/services/tools/StreamingToolExecutor.ts` | 560  | 流式工具并行执行器                        |
| `src/services/tools/toolOrchestration.ts`     | 207  | 非流式工具编排（串行/并行）               |

## 第一步：信息收集与 Prompt 组装

在 `queryLoop` 每次迭代的开头，prompt 由三层拼接而成：

### 1. System Prompt（固定部分，每次迭代不变）

```typescript
// context.ts — memoize 后全局缓存，只算一次
getSystemContext()  → { gitStatus, cacheBreaker }
getUserContext()    → { claudeMd, currentDate }

// query.ts:574 — 拼装
fullSystemPrompt = appendSystemContext(systemPrompt, systemContext)
// 结果: [...systemPrompt, "gitStatus: ...", "currentDate: ..."]
```

- `systemPrompt` 由上层 `QueryEngine` 传入，包含 CLAUDE.md 内容、工具描述、项目指令等
- `systemContext` 是 git 状态（分支、status、最近 5 个 commit）—— memoize，整个会话只算一次
- `getUserContext()` 是 CLAUDE.md 文件内容 + 当前日期

### 2. User Context（作为第一条 user message 注入）

```typescript
// query.ts:827 — 在消息数组最前面插入一条 meta message
messages = prependUserContext(messagesForQuery, userContext)
```

实际效果是在对话历史最前面插入一条 `<system-reminder>` 包裹的用户上下文（CLAUDE.md、日期），作为第一条 user message。

### 3. 消息历史（对话历史 + 工具结果）

```typescript
// query.ts:1932 — 每轮迭代结束后拼接
state = {
  messages: messagesForQuery.concat(assistantMessages, toolResults),
  //              ↑ 上一轮历史     ↑ 这轮模型输出  ↑ 这轮工具结果
}
```

## 第二步：API Provider 选择

Provider 选择**不在 query 循环内**，而是在 `queryModel()` → `getAnthropicClient()` 中根据环境变量决定：

```typescript
// query.ts:826-875 — deps.callModel 就是 queryModelWithStreaming
for await (const message of deps.callModel({
  messages: prependUserContext(messagesForQuery, userContext),
  systemPrompt: fullSystemPrompt,
  tools: toolUseContext.options.tools,
  options: { model: currentModel, ... },
}))
```

`queryModelWithStreaming` → `queryModel` 内部按 provider 分发：

```
queryModel() (1038行)
  ├─ getAPIProvider() === 'openai'  → queryModelOpenAI()   return
  ├─ getAPIProvider() === 'gemini'  → queryModelGemini()   return
  ├─ getAPIProvider() === 'grok'    → queryModelGrok()     return
  └─ 默认 → withRetry() → anthropic.beta.messages.create({stream:true})
```

Provider 选择完全由环境变量驱动，循环不感知。模型降级（Opus→Sonnet）在 `withRetry` 内通过 `FallbackTriggeredError` 触发，`queryLoop` 捕获后切换 `currentModel` 并 `continue`。

## 第三步：流式请求与响应处理

### A. 流式事件实时处理（claude.ts 的 SSE 循环）

```
message_start       → 记录 TTFB，初始化 partialMessage
content_block_start → 创建 contentBlock（text/tool_use/thinking）
content_block_delta → 累加到 streamingDeltas[index]（数组，避免 O(n²)）
content_block_stop  → deltas.join('') → 构建 AssistantMessage → yield
message_delta       → 更新 usage/stop_reason
message_stop        → 流结束
```

每个 `content_block_stop` 都会 `yield` 一条 `AssistantMessage`。

### B. 工具并行启动（query.ts 的消费循环）

```typescript
// query.ts:1006-1047 — 收到 assistant message 时
if (message.type === 'assistant') {
  assistantMessages.push(message)

  // 提取 tool_use blocks
  const msgToolUseBlocks = content.filter(c => c.type === 'tool_use')
  if (msgToolUseBlocks.length > 0) {
    toolUseBlocks.push(...msgToolUseBlocks)
    needsFollowUp = true  // ← 标记需要继续循环
  }

  // 流式工具执行：不等流结束，立即启动工具
  if (streamingToolExecutor) {
    for (const toolBlock of msgToolUseBlocks) {
      streamingToolExecutor.addTool(toolBlock, assistantMessage)
    }
  }

  // 立即取出已完成的工具结果 yield 出去
  for (const result of streamingToolExecutor.getCompletedResults()) {
    yield result.message
    toolResults.push(...)
  }
}
```

## 第四步：工具执行

### 流式执行器（StreamingToolExecutor）

流式执行器的并发控制：

```
StreamingToolExecutor.addTool(block)
  ├─ 判断 isConcurrencySafe（tool.isConcurrencySafe(input)）
  ├─ 加入队列，状态 = 'queued'
  └─ processQueue() → canExecuteTool?
       ├─ 无工具在执行 → 立即启动
       ├─ 当前工具并发安全 && 正在执行的工具都并发安全 → 并行启动
       └─ 非并发安全 → 等待（break，等前面的完成后 finally 触发 processQueue）
```

并发安全由每个工具的 `isConcurrencySafe(input)` 方法决定。Bash 工具通常是**非并发安全**的（文件操作可能冲突），Read/Grep/WebSearch 是**并发安全**的。

### 非流式模式（runTools）

等流结束后才批量执行：

```
runTools()
  ├─ partitionToolCalls() → [{isConcurrencySafe, blocks}]
  ├─ 并发安全 batch → runToolsConcurrently()（Promise.all，最大10并发）
  └─ 非并发 batch  → runToolsSerially()（逐个执行）
```

### Bash 错误级联

当 Bash 工具报错时，会通过 `siblingAbortController` 中止所有正在执行的兄弟工具：

```typescript
// StreamingToolExecutor.ts:386-389
if (tool.block.name === BASH_TOOL_NAME) {
  this.hasErrored = true
  this.erroredToolDescription = this.getToolDescription(tool)
  this.siblingAbortController.abort('sibling_error')
}
```

## 第五步：循环终止判定

```typescript
// query.ts:1250-1548 — 流结束后
if (!needsFollowUp) {
  // 没有工具调用 → 模型已经给出最终回答

  // 但还要检查各种恢复路径：
  ├─ prompt_too_long 恢复（context collapse → reactive compact）
  ├─ max_output_tokens 恢复（escalate → 64k，或注入恢复消息继续）
  ├─ stop hook 拦截（阻止继续）
  ├─ token budget 检查
  └─ 全部通过 → return { reason: 'completed' }
}
```

如果 `needsFollowUp = true`（有工具调用），流结束后进入工具执行阶段，执行完后拼接新状态继续循环。

## 完整数据流示例

```
用户输入 "修复这个 bug"
    │
    ▼
queryLoop iteration 1:
    │
    ├─ 上下文预处理
    │   messagesForQuery = [userInput]
    │   fullSystemPrompt = [CLAUDE.md, gitStatus, currentDate, 工具描述...]
    │   prependUserContext → [userContextMsg, userInput]
    │
    ├─ callModel({messages, systemPrompt, tools, model})
    │   │
    │   ├─ queryModel() → 构建 params → withRetry() → API streaming
    │   │
    │   └─ SSE 事件流:
    │       message_start → ...
    │       content_block_start(text) → delta → stop
    │         → yield AssistantMessage("让我看看代码...")
    │       content_block_start(tool_use: FileRead) → delta → stop
    │         → yield AssistantMessage(FileReadTool)
    │       content_block_start(tool_use: Bash) → delta → stop
    │         → yield AssistantMessage(BashTool)
    │       content_block_start(tool_use: FileEdit) → delta → stop
    │         → yield AssistantMessage(FileEditTool)
    │       message_delta → usage update
    │       message_stop
    │
    ├─ 流式执行（流期间已启动 FileRead）
    │   getCompletedResults() → yield FileRead 的结果
    │   Bash 和 FileEdit 还在队列中
    │
    ├─ needsFollowUp = true → 进入工具执行
    │
    ├─ getRemainingResults() → 等待 Bash 完成 → yield 结果
    │                         → FileEdit 启动 → 完成 → yield 结果
    │
    └─ state = { messages: [userCtx, input, assistant, fileRead_result,
                             bash_result, fileEdit_result] }
      turnCount = 2
      continue → 回到 while(true)
         │
         ▼
queryLoop iteration 2:
    │
    ├─ 上下文预处理（autocompact 检查等）
    │
    ├─ callModel({messages: [...全部历史...], ...})
    │   └─ SSE: yield AssistantMessage("已修复，改动如下...")
    │       （没有 tool_use blocks）
    │
    ├─ needsFollowUp = false → 不执行工具
    │
    └─ return { reason: 'completed' }
```

## 核心设计要点

1. **`query()` 是 AsyncGenerator** — 每一步都 `yield` 消息给调用者（REPL 屏幕），实现实时 UI 更新
2. **`State` 对象在迭代间传递** — 7 个 `continue` 站点都用 `state = { ... }` 而非逐字段赋值
3. **流式执行 vs 非流式执行** — `StreamingToolExecutor` 在 `content_block_stop` 时就启动工具；`runTools` 在流结束后才批量执行。两者产出相同格式的 `MessageUpdate`
4. **并发控制由工具自己决定** — `isConcurrencySafe(input)` 是每个工具的方法，Bash 默认不安全，Read 默认安全
5. **恢复路径丰富** — max_output_tokens（escalate → 64k → 注入恢复消息）、prompt_too_long（collapse drain → reactive compact）、模型降级（529 连续 3 次 → FallbackTriggeredError）
