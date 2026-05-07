# 记忆系统

## 总览

Claude Code 的记忆系统由两部分组成：

1. **手动记忆**（CLAUDE.md）— 用户编写的项目指令，随项目提交
2. **自动记忆**（Auto Memory）— Claude 自动提取的持久化知识，跨会话保留

```
记忆系统
├─ CLAUDE.md（手动）
│   ├─ 项目级: ./CLAUDE.md, ./.claude/CLAUDE.md
│   ├─ 规则级: ./.claude/rules/*.md（带 frontmatter glob）
│   ├─ 本地级: ./CLAUDE.local.md（.gitignore）
│   └─ 用户级: ~/.claude/CLAUDE.md
│
└─ Auto Memory（自动）
    ├─ 存储目录: ~/.claude/projects/<git-root>/memory/
    ├─ 索引文件: MEMORY.md
    ├─ 分类文件: user.md, feedback.md, project.md, reference.md
    └─ 提取时机: 每轮 query 循环结束时
```

## 核心文件

| 文件                                              | 职责                           |
| ------------------------------------------------- | ------------------------------ |
| `src/utils/claudemd.ts`                           | CLAUDE.md 发现、加载、合并     |
| `src/context.ts`                                  | 记忆注入到 system prompt       |
| `src/memdir/paths.ts`                             | Auto Memory 路径解析、开关     |
| `src/memdir/memdir.ts`                            | MEMORY.md 索引管理、截断       |
| `src/memdir/memoryScan.ts`                        | 记忆文件扫描和 manifest        |
| `src/memdir/findRelevantMemories.ts`              | 按相关性检索记忆               |
| `src/memdir/memoryTypes.ts`                       | 记忆类型定义、frontmatter 格式 |
| `src/services/extractMemories/extractMemories.ts` | 自动提取记忆的核心逻辑         |
| `src/services/extractMemories/prompts.ts`         | 提取 prompt 模板               |
| `src/utils/attachments.ts`                        | 记忆 prefetch 和注入           |

## CLAUDE.md 手动记忆

### 发现优先级

从高到低：

```
1. /etc/claude-code/CLAUDE.md         → 管理员全局指令
2. ~/.claude/CLAUDE.md                 → 用户全局指令
3. ./CLAUDE.md                        → 项目指令（可提交）
4. ./.claude/CLAUDE.md                → 项目私有指令
5. ./.claude/rules/*.md               → 条件规则（带 frontmatter）
6. ./CLAUDE.local.md                  → 本地私有（不提交）
```

### 条件规则

`.claude/rules/` 下的文件支持 frontmatter glob 匹配：

```markdown
---
paths:
  - "src/services/**"
  - "*.test.ts"
---

仅当编辑匹配路径的文件时，这些规则才会注入。
```

### @include 指令

记忆文件可以引用其他文件：

```markdown
# CLAUDE.md
@docs/architecture.md
@./specs/api-spec.md
```

### 注入方式

```typescript
// context.ts:155-189 — memoize，整个会话只算一次
getUserContext() → {
  claudeMd: getClaudeMds(getMemoryFiles()),  // 所有 CLAUDE.md 内容合并
  currentDate: "Today's date is 2026-05-06.",
}

// query.ts:827 — 作为第一条 user message 注入
messages = prependUserContext(messages, userContext)
// → <system-reminder> 包裹的 CLAUDE.md 内容
```

## Auto Memory 自动记忆

### 启用条件

```typescript
isAutoMemoryEnabled():
  1. CLAUDE_CODE_DISABLE_AUTO_MEMORY 未设为 true
  2. 不是 --bare 模式
  3. CCR 环境需要有持久化存储
  4. settings.json 中 autoMemoryEnabled 未设为 false
  5. 默认：启用
```

### 存储结构

```
~/.claude/projects/<sanitized-git-root>/memory/
├── MEMORY.md              ← 索引文件（200 行上限）
├── user_role.md           ← 用户角色/技能记忆
├── feedback_prefs.md      ← 用户偏好/纠正记忆
├── project_ctx.md         ← 项目上下文记忆
├── reference_links.md     ← 外部系统指针
└── logs/                  ← 日期日志（Kairos 模式）
    └── 2026/05/2026-05-06.md
```

### 记忆文件格式

```markdown
---
name: 用户角色
description: 用户的职业背景和技能水平
type: user
---

用户是一名后端工程师，有 10 年 Go 经验，但第一次接触 React。
当解释前端概念时，用后端类比来帮助理解。
```

**四种记忆类型**：

| 类型        | 用途       | 示例                            |
| ----------- | ---------- | ------------------------------- |
| `user`      | 用户画像   | 角色、技能、偏好                |
| `feedback`  | 纠正/偏好  | "不要在回复末尾加总结"          |
| `project`   | 项目上下文 | "这个模块正在重构"              |
| `reference` | 外部系统   | "bug 追踪在 Linear INGEST 项目" |

### MEMORY.md 索引

```markdown
# 学习笔记

## 参考

- [用户角色](user_role.md) — 后端工程师，Go 专家，React 新手
- [编码偏好](feedback_prefs.md) — 不要加总结，用简洁风格
- [项目背景](project_ctx.md) — 正在重构认证模块
```

索引行上限 200 行、25KB。每行格式：`- [Title](file.md) — one-line hook`

### 提取时机

```typescript
// stopHooks.ts — 每轮 query 循环结束时
if (feature('EXTRACT_MEMORIES') && isAutoMemoryEnabled()) {
  await extractMemories(messages, toolUseContext)
}
```

### 提取流程

```
query 循环结束（needsFollowUp = false）
    ↓
extractMemories(messages, toolUseContext)
    ├─ 检查：主 agent 是否已经写了记忆？（互斥）
    ├─ 扫描最近的 assistant/user 消息
    ├─ 检查已有记忆文件，避免重复
    ├─ 用 forked agent（runForkedAgent）分析内容
    │   └─ 共享父会话的 prompt cache（零额外成本）
    ├─ 生成记忆文件（带 frontmatter）
    └─ 更新 MEMORY.md 索引
```

### 互斥机制

```
extractMemories 检查 hasMemoryWritesSince:
  ├─ 主 agent 写了记忆 → 跳过（不重复提取）
  └─ 主 agent 没写 → 提取（捕获遗漏）
```

### 不保存的内容

```
代码模式、架构、文件路径、项目结构
Git 历史、调试方案、修复配方
当前会话的临时状态
任何可以从代码推导的信息
```

## 记忆 Prefetch

### 工作原理

```typescript
// query.ts:416 — 在 queryLoop 开始时启动（不阻塞）
using pendingMemoryPrefetch = startRelevantMemoryPrefetch(
  state.messages,
  state.toolUseContext,
)

// query.ts:1807-1829 — 工具执行后消费
if (pendingMemoryPrefetch?.settledAt !== null) {
  const memoryAttachments = filterDuplicateMemoryAttachments(
    await pendingMemoryPrefetch.promise,
    toolUseContext.readFileState,
  )
  for (const mem of memoryAttachments) {
    yield createAttachmentMessage(mem)
    toolResults.push(msg)
  }
}
```

### 相关性匹配

```typescript
// findRelevantMemories.ts
function findRelevantMemories(userMessage: string, memoryFiles: MemoryFile[]) {
  // 基于用户消息内容匹配记忆文件
  // 返回最相关的记忆（最多 5 个）
  return rankedMemories.slice(0, 5)
}
```

### 限制

```typescript
MAX_MEMORY_LINES = 200      // 每个记忆文件最多 200 行
MAX_MEMORY_BYTES = 4096    // 每个记忆文件最多 4KB
```

超限文件显示截断警告和完整内容链接。

## 记忆写入权限

Auto Memory 目录有特殊的写入权限：

```typescript
// memdir/paths.ts
isAutoMemPath(absolutePath) → boolean

// 只有在 auto memory 目录内的路径才允许写入
// FileWriteTool / FileEditTool 检查此路径
// 安全：项目设置中的 projectSettings 被排除（防止恶意仓库写入 ~/.ssh 等）
```

## 数据流总结

```
[会话开始]
    ↓
context.ts: getUserContext()
    ├─ getMemoryFiles() → 发现所有 CLAUDE.md
    ├─ getClaudeMds() → 合并内容
    └─ 缓存到 setCachedClaudeMdContent()
    ↓
query.ts: prependUserContext(messages, userContext)
    → CLAUDE.md 注入为第一条 user message
    ↓
query.ts: startRelevantMemoryPrefetch()
    ├─ 异步读取 Auto Memory 文件
    ├─ 按相关性排序
    └─ 等待消费（不阻塞 API 调用）
    ↓
[query 循环结束]
    ↓
stopHooks: extractMemories()
    ├─ 扫描对话消息
    ├─ Forked agent 分析内容
    ├─ 写入记忆文件 + 更新索引
    └─ yield createMemorySavedMessage()
    ↓
[下一次会话]
    ↓
同上 → 读取更新后的记忆
```
