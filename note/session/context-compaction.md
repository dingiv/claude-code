# 上下文压缩系统

## 总览

Claude Code 用多层压缩策略防止对话超出模型上下文窗口。按粒度和触发时机分为五个层级：

```
压缩策略（按激进程度排序）
├─ Micro-compact    ← 每轮迭代，清空旧工具结果内容（最快，无 API 调用）
├─ Snip compact     ← 用户主动标记删除消息
├─ Auto-compact     ← token 阈值触发，Haiku 生成摘要
├─ Context collapse ← 细粒度段落级管理
└─ Reactive compact ← prompt-too-long 紧急恢复
```

## 核心文件

| 文件 | 职责 |
|------|------|
| `src/services/compact/autoCompact.ts` | 自动压缩：阈值检测、Haiku 摘要、边界消息 |
| `src/services/compact/compact.ts` | 核心压缩逻辑：compactConversation、buildPostCompactMessages |
| `src/services/compact/microCompact.ts` | 微压缩：时间/计数触发，清空旧工具结果 |
| `src/services/compact/cachedMicrocompact.ts` | 缓存微压缩：通过 API cache editing 删除 |
| `src/services/compact/snipCompact.ts` | 历史裁剪：标记删除消息 |
| `src/services/compact/reactiveCompact.ts` | 反应式压缩：prompt-too-long 恢复 |
| `src/services/contextCollapse/` | 上下文折叠：细粒度管理 |
| `src/services/compact/sessionMemoryCompact.ts` | 会话记忆压缩（实验性） |

## 执行顺序

在 `queryLoop` 每轮迭代中，压缩管道按以下顺序执行：

```typescript
// query.ts — 每轮迭代的预处理阶段
messagesForQuery = await applyToolResultBudget(messages)    // 工具结果大小限制
messagesForQuery = snipCompactIfNeeded(messagesForQuery)      // ① Snip
messagesForQuery = await microcompact(messagesForQuery)       // ② Micro-compact
messagesForQuery = await contextCollapse(messagesForQuery)     // ③ Context collapse
messagesForQuery = await autocompact(messagesForQuery)        // ④ Auto-compact（条件触发）
```

## Auto-Compact

### 触发阈值

```typescript
getAutoCompactThreshold(model) = effectiveContextWindow - autocompactBuffer

// 缓冲区大小按上下文窗口缩放
getAutocompactBufferTokens(model):
  800K+ 上下文 → 50K buffer
  400K+ 上下文 → 30K buffer
  默认           → 13K buffer (AUTOCOMPACT_BUFFER_TOKENS)

// 预测性压缩：估计本轮增长是否超出
estimateMaxTurnGrowth(model) = maxOutputTokens + 15_000
```

### 工作流程

```
tokenCountWithEstimation(messages) >= threshold
    ↓
shouldAutoCompact() → true
    ↓
autoCompactIfNeeded()
    ├─ 优先尝试 sessionMemoryCompaction（使用已有摘要）
    └─ 失败 → compactConversation()
        ├─ 提取要保留的最近消息
        ├─ 用 Haiku 总结旧消息
        ├─ 恢复关键上下文（打开的文件、skill、plan）
        └─ 返回 CompactionResult
    ↓
buildPostCompactMessages(result)
    = [boundaryMarker, ...summaryMessages, ...keptMessages, ...attachments, ...hookResults]
    ↓
yield boundaryMessage + summaryMessages → 替换 messagesForQuery
```

### 边界消息

```typescript
// Compaction 后插入的标记消息
{
  type: 'system',
  subtype: 'compact_boundary',
  compactMetadata: {
    trigger: 'auto' | 'manual',
    preTokens: 185000,           // 压缩前的 token 数
    messagesSummarized: 42,      // 被摘要的消息数
    userContext: "...",          // 压缩时的 userContext
  },
  logicalParentUuid: 'uuid'     // 压缩前最后一条消息的 ID
}
```

边界消息的作用：
- 告诉模型"这里有上下文断裂"
- 防止 API 跨边界读取（thinking block 签名不匹配）
- 恢复时知道从哪里开始

### 断路器

```typescript
MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3

// 连续 3 次失败后停止尝试
tracking.consecutiveFailures++
if (tracking.consecutiveFailures >= 3) {
  // 不再重试，返回错误
}
```

### 跨轮状态追踪

```typescript
type AutoCompactTrackingState = {
  compacted: boolean          // 是否已压缩过
  turnCounter: number          // 压缩后的轮次数
  turnId: string               // 当前轮唯一 ID
  consecutiveFailures?: number // 连续失败次数
}
```

追踪状态通过 `state` 对象在迭代间传递。compact 后 `turnCounter++`，用于限制"compact 后快速轮"（避免反复 compact）。

## Micro-Compact

### 与 Auto-Compact 的区别

| | Auto-Compact | Micro-Compact |
|---|---|---|
| **触发** | Token 阈值 | 时间间隔 / 计数 |
| **操作** | Haiku 生成摘要 | 清空旧工具结果内容 |
| **API 调用** | 需要 | 不需要 |
| **信息损失** | 旧消息被摘要替代 | 工具结果内容被清空，结构保留 |
| **速度** | 慢（几秒） | 快（毫秒级） |

### 时间触发

```typescript
// 如果最近一条 assistant 消息距今超过阈值
if (timeSinceLastAssistantMessage > config.timeThresholdMinutes) {
  // 清空除最近 N 条外的所有可压缩工具结果
  for (const toolResult of compactableToolResults) {
    if (toolResult.age > keepRecent) {
      toolResult.content = TIME_BASED_MC_CLEARED_MESSAGE  // "[Old tool result content cleared]"
    }
  }
}
```

### 可压缩工具

```typescript
const COMPACTABLE_TOOLS = new Set([
  'FileReadTool', 'ShellTool', 'PowerShellTool',
  'GrepTool', 'GlobTool',
  'WebSearchTool', 'WebFetchTool',
  'FileEditTool', 'FileWriteTool',
])
```

只有这些工具的结果可以被清空（都是"读取型"或"历史型"信息）。FileEdit 的内容会被清空但 `file_path` 等元数据保留。

### Cached Micro-Compact

Ant-only 功能，使用 API 的 cache editing 能力：

```typescript
// 不清空内容，而是通过 API 删除缓存
// 优势：不会破坏 prompt cache
pendingCacheEdits = {
  edits: [
    { cache_reference: tool_use_id, cache_edit_type: 'delete' }
  ]
}

// 在 API 请求中发送 cache_edits
// API 会在服务端删除这些 token，不影响已缓存的前缀
```

## Snip Compact

用户通过 Snip 工具主动标记消息删除：

```typescript
// Snip 工具在消息上设置 snip 标记
message._snip = true

// snipCompactIfNeeded 在 API 请求前过滤
function snipCompactIfNeeded(messages) {
  const toRemove = messages.filter(m => m._snip)
  const remaining = messages.filter(m => !m._snip)

  if (toRemove.length > 0) {
    // 插入边界消息记录删除了什么
    const boundary = createSnipBoundaryMessage(toRemove.map(m => m.uuid))
    return { messages: [boundary, ...remaining], tokensFreed: estimated }
  }
}
```

### 主动裁剪

消息数量超过 30 条时，Snip 会建议用户使用裁剪工具。

### 内存保护

In-memory 消息存储超过 150MB 时，proactive truncation 会自动裁剪最旧的消息。

## Reactive Compact

当 API 返回 "prompt too long" 错误时的紧急恢复：

```typescript
// query.ts:1258-1364
if (isWithheld413) {
  // 1. 先尝试 context collapse drain（保留细粒度上下文）
  if (contextCollapse.recoverFromOverflow(messages)) {
    continue  // 重试
  }

  // 2. 再尝试 reactive compact（全文摘要）
  if (reactiveCompact.tryReactiveCompact()) {
    continue  // 重试
  }

  // 3. 全部失败 → 返回错误
  yield errorMessage
  return { reason: 'prompt_too_long' }
}
```

## max_output_tokens 恢复

当模型输出被截断时的恢复路径：

```
max_output_tokens hit
    ├─ 第一次：escalate 到 64K tokens（如果之前用的是默认 8K）
    ├─ 2-3 次：注入恢复消息
    │   "Output token limit hit. Resume directly — no apology.
    │    Pick up mid-thought if that is where the cut happened."
    └─ 3 次后：放弃，返回错误
```

## Compact 后的上下文恢复

压缩后需要恢复关键上下文，否则模型不知道"当前在做什么"：

```typescript
// compact.ts — post-compact 恢复
POST_COMPACT_TOKEN_BUDGET = 50_000
POST_COMPACT_MAX_TOKENS_PER_FILE = 5_000

// 恢复内容
├─ 当前打开的文件（从 readFileState 获取）
├─ 活跃的 skill 描述
├─ 活跃的 plan（从 agent context 获取）
├─ 最近使用的工具摘要
└─ 用户上下文（从压缩前的 userContext 保存）
```

## Token 预算系统

```typescript
// query.ts — 预测性压缩
const currentTokens = tokenCountWithEstimation(messages)
const estimatedGrowth = estimateMaxTurnGrowth(model)
const predictiveThreshold = getEffectiveContextWindowSize(model) - estimatedGrowth

if (currentTokens > predictiveThreshold) {
  // 在 API 调用前就压缩，避免 413 错误
  await autocompact(messages)
}
```

## 数据流总结

```
queryLoop iteration:
    │
    ├─ applyToolResultBudget()     → 限制单个工具结果大小
    ├─ snipCompactIfNeeded()       → 删除用户标记的消息
    ├─ microcompact()              → 清空旧工具结果（快）
    ├─ contextCollapse()           → 细粒度段落管理
    ├─ autocompact()               → Haiku 摘要（慢，条件触发）
    │
    ├─ API 调用
    │   ├─ 成功 → 继续工具执行
    │   ├─ prompt_too_long → reactive compact → 重试
    │   └─ max_output_tokens → escalate/recovery → 重试
    │
    └─ state.messages = postCompactMessages
```
