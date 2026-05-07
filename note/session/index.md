# 会话管理

## 架构总览

会话管理由三层组成，每层有明确的职责边界：

```
REPL.tsx（UI 层）
    ↓ 用户输入
QueryEngine（会话控制层）
    ↓ submitMessage()
query() → queryLoop()（Agent 循环层）
```

## 核心文件

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/QueryEngine.ts` | 1367 | 会话生命周期管理，submitMessage 入口 |
| `src/screens/REPL.tsx` | — | 终端 UI，驱动输入和显示 |
| `src/bootstrap/state.ts` | — | 会话级全局状态（session ID、CWD、token 计数） |
| `src/state/AppState.tsx` | — | 中心状态类型和 Context Provider |
| `src/state/store.ts` | — | Zustand 风格 Store |
| `src/utils/sessionStorage.ts` | — | 消息持久化到 JSONL transcript |
| `src/utils/sessionRestore.ts` | — | 会话恢复（--resume） |
| `src/utils/handlePromptSubmit.ts` | — | 输入处理协调 |
| `src/utils/QueryGuard.ts` | — | 查询并发控制和取消 |

## QueryEngine：会话控制器

QueryEngine 是**每个会话一个实例**，拥有会话的完整生命周期。

### 核心状态

```typescript
class QueryEngine {
  private mutableMessages: Message[]      // 对话历史
  private abortController: AbortController // 取消控制
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录
  private totalUsage: NonNullableUsage    // 累计 token 用量
  private readFileState: FileStateCache   // 文件读取缓存（跨轮次）
}
```

### submitMessage() 流程

```typescript
async *submitMessage(prompt: string | ContentBlockParam[]) {
  // ① 获取系统 Prompt 三件套
  const { defaultSystemPrompt, userContext, systemContext } =
    await fetchSystemPromptParts({ tools, mainLoopModel, mcpClients })

  // ② 组装最终 SystemPrompt
  const systemPrompt = asSystemPrompt([
    ...(customPrompt ?? defaultSystemPrompt),
    ...(memoryMechanicsPrompt ?? []),
    ...(appendSystemPrompt ?? []),
  ])

  // ③ 处理用户输入（斜杠命令、附件、图片等）
  const { messages, shouldQuery, allowedTools } =
    await processUserInput({ input: prompt, mode: 'prompt' })
  this.mutableMessages.push(...messages)

  // ④ 持久化用户消息到 transcript（在 API 调用之前！）
  await recordTranscript(messages)

  // ⑤ 构建 ToolUseContext
  const toolUseContext = { options: { tools, mainLoopModel, ... }, ... }

  // ⑥ 调用 query() Agent 循环
  for await (const message of query({
    messages: this.mutableMessages,
    systemPrompt, userContext, systemContext,
    canUseTool, toolUseContext, ...
  })) {
    // 收集消息、更新状态、yield 给上层
    this.mutableMessages.push(normalizeMessage(message))
    yield toSDKMessage(message)
  }
}
```

**关键设计：先持久化再调 API**。即使用户在 API 响应前杀掉进程，transcript 也已经包含用户消息，`--resume` 可以恢复。

## 会话 ID 与全局状态

```typescript
// bootstrap/state.ts — 模块级单例
let sessionId: string = randomUUID()
getSessionId()         → 当前会话 UUID
regenerateSessionId() → 创建新 ID（/clear 时调用）
switchSession(id)     → 原子切换到另一个会话
```

全局状态通过模块级变量实现，不是 class 实例：

```typescript
// bootstrap/state.ts 中的关键状态
let sessionId: string
let cwd: string
let totalInputTokens: number
let totalOutputTokens: number
let totalCacheReadInputTokens: number
let totalCacheCreationInputTokens: number
// ... Beta header latches（fastMode, afkMode, cacheEditing）
```

## 消息持久化（Transcript）

### 存储格式

消息以 JSONL 格式存储在会话目录下：

```typescript
// sessionStorage.ts
recordTranscript(messages)  // 追加写入 JSONL 文件
flushSessionStorage()       // 强制刷盘
```

### 恢复流程

```
--resume
    ↓
getLastSessionLog() → 读取最新 transcript
    ↓
restoreSessionStateFromLog()
    ├─ 恢复文件历史快照（fileHistory）
    ├─ 恢复 attribution commits
    ├─ 恢复 TodoWrite 状态
    ├─ 恢复 agent 设置和 model 覆盖
    └─ 重建消息数组
```

## REPL → QueryEngine 的输入流

```
用户输入 "修复 bug"
    ↓
handlePromptSubmit()
    ├─ 处理 @引用、粘贴内容、图片
    ├─ 斜杠命令？→ 立即执行（/clear, /model, /compact 等）
    └─ 普通输入 → 排队
    ↓
executeUserInput()
    ↓
QueryEngine.submitMessage(prompt)
    ↓
processUserInput() → 处理斜杠命令、生成附件消息
    ↓
query() → queryLoop() → API 调用
```

### QueryGuard：并发控制

QueryGuard 防止多个查询同时运行：

```typescript
// QueryGuard.ts
class QueryGuard {
  isQueryActive: boolean  // 当前是否有查询在运行
  startQuery()            // 开始查询（检查并发）
  endQuery()              // 结束查询
}
```

用户发送新消息时，如果上一个查询还在运行：
- **普通输入**：排队等待
- **ESC 中断**：abortController.abort()，等待当前查询结束后处理

## 数据流总结

```
[用户输入] → REPL → handlePromptSubmit → processUserInput
                                              ↓
                                        [斜杠命令?] → 立即执行
                                              ↓ (否)
                                        QueryEngine.submitMessage()
                                              ↓
                                        fetchSystemPromptParts()
                                        recordTranscript() ← 先持久化
                                              ↓
                                        query() → queryLoop()
                                              ↓ yield
                                        [AssistantMessage / ToolResult]
                                              ↓
                                        mutableMessages.push()
                                        yield toSDKMessage()
                                              ↓
                                        REPL 渲染到终端
```
