# Prompt 拼装

## 总览

Claude Code 的 prompt 由三部分拼接，每部分有独立的收集时机和缓存策略：

```
最终 API 请求
├─ system: SystemPrompt blocks（带 cache_control）
├─ messages[0]: UserContext meta message
└─ messages[1..n]: 对话历史 + 工具结果
```

## 核心文件

| 文件 | 职责 |
|------|------|
| `src/utils/queryContext.ts` | fetchSystemPromptParts() — 三件套的统一入口 |
| `src/constants/prompts.ts` | getSystemPrompt() — 核心系统 prompt 的段落组装 |
| `src/context.ts` | getSystemContext() / getUserContext() — git 状态和 CLAUDE.md |
| `src/utils/claudemd.ts` | CLAUDE.md 文件发现、加载、合并 |
| `src/utils/api.ts` | prependUserContext / appendSystemContext / toolToAPISchema |
| `src/services/api/claude.ts` | buildSystemPromptBlocks / addCacheBreakpoints |
| `src/utils/messages.ts` | normalizeMessagesForAPI |

## 三件套收集

### 1. defaultSystemPrompt（核心指令）

```typescript
// queryContext.ts:44-74
async function fetchSystemPromptParts({ tools, mainLoopModel, mcpClients }) {
  return {
    defaultSystemPrompt: getSystemPrompt(),
    userContext: await getUserContext(),
    systemContext: await getSystemContext(),
  }
}
```

`getSystemPrompt()` 在 `src/constants/prompts.ts` 中组装，由多个**段落**拼接而成：

```
[角色定义]
"You are Claude Code, ..."
[行为规则]
- 如何使用工具
- 如何格式化输出
- 代码修改规范
[环境信息]
- 当前日期
- 操作系统
[特性指令]
- MCP 工具使用说明（如果启用）
- Proactive features（如果启用）
- Thinking 模式说明
[工具描述]
59+ 个工具的 JSON schema
```

### 2. userContext（CLAUDE.md + 日期）

```typescript
// context.ts:155-189 — memoize，整个会话只算一次
getUserContext() → {
  claudeMd,        // CLAUDE.md 文件内容
  currentDate,     // "Today's date is 2026-05-06."
}
```

**CLAUDE.md 发现优先级**（从高到低）：

```
1. Managed memory:    /etc/claude-code/CLAUDE.md
2. User memory:       ~/.claude/CLAUDE.md
3. Project memory:    ./CLAUDE.md
                      ./.claude/CLAUDE.md
                      ./.claude/rules/*.md
4. Local memory:      ./CLAUDE.local.md
```

支持 `@include` 指令和 frontmatter 条件规则（glob 匹配文件路径）。

### 3. systemContext（Git 状态）

```typescript
// context.ts:116-150 — memoize，整个会话只算一次
getSystemContext() → {
  gitStatus,      // branch, mainBranch, status, recent commits, gitUser
  cacheBreaker,   // ant-only 调试用
}
```

## 拼装流程

### Step 1: QueryEngine 组装 SystemPrompt

```typescript
// QueryEngine.ts:335-339
const systemPrompt = asSystemPrompt([
  ...(customPrompt ?? defaultSystemPrompt),
  ...(memoryMechanicsPrompt ?? []),
  ...(appendSystemPrompt ?? []),
])
```

### Step 2: queryLoop 合并 systemContext

```typescript
// query.ts:574
const fullSystemPrompt = appendSystemContext(systemPrompt, systemContext)
// 结果: [...systemPrompt, "gitStatus: ...", "currentDate: ..."]
```

### Step 3: 注入 userContext 为第一条消息

```typescript
// query.ts:827
messages = prependUserContext(messagesForQuery, userContext)
// 在消息数组最前面插入一条 meta message:
// <system-reminder>
// As you answer the user's questions, you can use the following context:
// # claudeMd
// ...
// # currentDate
// Today's date is 2026-05-06.
// </system-reminder>
```

### Step 4: 构建 API system blocks

```typescript
// claude.ts:3364-3388
system = buildSystemPromptBlocks(systemPrompt, enablePromptCaching, options)
```

将 `string[]` 转为 `{ type: 'text', text, cache_control? }[]`：
- 使用 `splitSysPromptPrefix()` 确定缓存边界
- 分为"稳定前缀"和"动态后缀"两部分

### Step 5: 添加 cache_control 标记

```typescript
// claude.ts:3213-3361
messages = addCacheBreakpoints(messages, enablePromptCaching, querySource)
```

**关键策略：每个请求只有一个 cache_control 标记**，放在最后一条消息上：

```typescript
const markerIndex = skipCacheWrite
  ? messages.length - 2  // skipCacheWrite 时放在倒数第二条
  : messages.length - 1  // 正常情况放在最后一条
```

原因：API 的 mycro 分页器会在每个 `cache_control` 位置保护 KV 页面。多标记 = 浪费缓存空间。

### Step 6: 消息规范化

```typescript
// messages.ts — normalizeMessagesForAPI()
messagesForAPI = normalizeMessagesForAPI(messages, filteredTools)
```

规范化处理：
- 附件消息冒泡到 tool_result 位置
- 剥离虚拟消息（display-only）
- 合并连续的 user 消息
- 处理 tool_search 相关字段
- 修复 tool_use / tool_result 配对

## 工具 Schema 转换

```typescript
// api.ts:119-266
toolToAPISchema(tool, { getToolPermissionContext, tools, agents, model, deferLoading })
```

将 Tool 对象转为 `BetaToolUnion`（Anthropic API 格式）：

```typescript
{
  name: "BashTool",
  description: "...",  // 由 tool.description(input) 动态生成
  input_schema: {       // Zod schema → JSON Schema
    type: "object",
    properties: { command: { type: "string" } },
    required: ["command"]
  },
  cache_control: { type: "ephemeral" },  // 工具列表末尾的工具带缓存
  ...(deferLoading && { defer_loading: true })  // MCP 工具动态加载
}
```

**动态描述**：工具的 `description()` 方法根据输入参数生成描述，而非静态字符串。例如 BashTool 会把具体命令包含在描述中。

## Prompt Caching 策略

### 缓存层级

```
System Prompt
├─ 稳定前缀（角色定义、规则）→ cache_control: { type: "ephemeral" }
└─ 动态后缀（日期、特性指令）→ 无缓存

工具 Schema 列表
└─ 最后一个工具 → cache_control: { type: "ephemeral" }

消息历史
└─ 最后一条消息 → cache_control: { type: "ephemeral" }
```

### 1h TTL 扩展

对 Ant/Subscriber 用户，支持 `ttl: "1h"` 长缓存：

```typescript
getCacheControl({ scope: 'global', querySource })
// → { type: 'ephemeral', ttl: '1h', scope: 'global' }
```

通过 GrowthBook allowlist 按 querySource 控制（如只给主线程和 SDK）。

### Global Cache Scope

当没有 MCP 工具（动态内容）时，启用全局缓存：

```
scope: 'global' → 整个 system prompt 在所有会话间共享缓存
```

有 MCP 工具时回退到 `system_prompt` 级别（MCP 工具描述是用户特定的，不能全局缓存）。

### Beta Header Latching

Fast Mode / AFK / Cache Editing 的 beta header 采用"一次开启，永久保持"策略：

```typescript
// 首次激活时 latch
if (!fastModeHeaderLatched && isFastMode) {
  fastModeHeaderLatched = true
  setFastModeHeaderLatched(true)
}
// 后续请求始终发送该 header，避免 mid-session 切换导致缓存失效
```

清除时机：`/clear` 和 `/compact` 时调用 `clearBetaHeaderLatches()`。

## 完整数据流

```
fetchSystemPromptParts()
    ├─ getSystemPrompt()     → 核心指令（段落拼接）
    ├─ getUserContext()      → CLAUDE.md + 日期（memoize）
    └─ getSystemContext()    → Git 状态（memoize）
    ↓
QueryEngine: systemPrompt = [customPrompt, memoryPrompt, appendPrompt]
    ↓
queryLoop: fullSystemPrompt = appendSystemContext(systemPrompt, systemContext)
    ↓
queryLoop: messages = prependUserContext(messages, userContext)
    ↓
queryModel:
    ├─ system = buildSystemPromptBlocks(fullSystemPrompt)
    ├─ tools = toolSchemas.map(toolToAPISchema)
    ├─ messages = normalizeMessagesForAPI(messages)
    ├─ messages = addCacheBreakpoints(messages)
    └─ params = { system, tools, messages, betas, ... }
    ↓
anthropic.beta.messages.create(params)
```
