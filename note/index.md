# 学习笔记

## 项目概览

Claude Code 是 Anthropic 官方 CLI 工具的逆向/反编译还原版本。基于 **Bun** 运行时 + **React/Ink** 终端 UI 构建，核心代码 10,000+ 行（顶层文件），含 59+ 内置工具和 7 种 API Provider 支持。

## 五层架构

| 层次           | 核心文件                       | 行数 | 职责                                      |
| -------------- | ------------------------------ | ---- | ----------------------------------------- |
| **交互层**     | `src/screens/REPL.tsx`         | —    | 终端 UI、用户输入、消息展示（React/Ink）  |
| **编排层**     | `src/QueryEngine.ts`           | 1367 | 多轮对话管理、会话持久化、成本追踪        |
| **核心循环层** | `src/query.ts`                 | 1945 | Agentic Loop：发请求→拿响应→执行工具→循环 |
| **工具层**     | `src/tools.ts` + `src/Tool.ts` | 1225 | 59+ 工具定义、权限校验、MCP 扩展          |
| **通信层**     | `src/services/api/claude.ts`   | —    | 7 种 Provider 流式通信                    |

## 核心数据流（Agentic Loop）

```
用户输入 → REPL → QueryEngine → query() 循环:
  ① 上下文预处理管道：applyToolResultBudget → snipCompact → microcompact → contextCollapse → autocompact
  ② 流式 API 调用：deps.callModel() → AsyncGenerator<StreamEvent | Message>
  ③ 工具执行：StreamingToolExecutor（并行）或 runTools（串行）
  ④ 终止/继续判定：needsFollowUp ? continue : return { reason }
```

## 入口与启动流程

| 入口         | 文件                              | 说明                                    |
| ------------ | --------------------------------- | --------------------------------------- |
| CLI 启动     | `src/entrypoints/cli.tsx` (378行) | 注入 feature() polyfill、MACRO 全局变量 |
| 命令定义     | `src/main.tsx` (5674行)           | Commander.js 解析参数，注册所有子命令   |
| 一次性初始化 | `src/entrypoints/init.ts`         | 遥测配置、信任对话框                    |
| 管道模式     | `src/main.tsx` `-p` flag          | `echo "say hello" \| bun run dev -p`    |

### CLI 快速路径（按优先级）

`cli.tsx` 的 `main()` 函数依次判断：
1. `--version` / `-v` — 零模块加载
2. `--dump-system-prompt` — feature-gated
3. `--claude-in-chrome-mcp` / `--chrome-native-host`
4. `--computer-use-mcp` — 独立 MCP server 模式
5. `--daemon-worker=<kind>` — feature-gated (DAEMON)
6. `remote-control` / `rc` / `bridge` — feature-gated (BRIDGE_MODE)
7. `daemon` [subcommand] — feature-gated (DAEMON)
8. `ps` / `logs` / `attach` / `kill` / `--bg` — feature-gated (BG_SESSIONS)
9. `new` / `list` / `reply` — Template job 命令
10. 默认路径：加载 `main.tsx` 启动完整 CLI

## Agentic Loop 详解（`src/query.ts`）

`query()` 是一个 `AsyncGenerator`，通过 `State` 类型（10 个字段）在迭代间传递状态：

```typescript
// 简化的循环结构
while (true) {
  // 1. 上下文预处理
  applyToolResultBudget(state)
  snipCompact(state)
  microcompact(state)
  contextCollapse(state)
  autocompact(state)

  // 2. 流式 API 调用
  const { assistantMessages, toolUseBlocks } = await callModel(state)

  // 3. 工具执行
  const toolResults = await executeTools(toolUseBlocks)

  // 4. 判断是否继续
  if (!needsFollowUp) return { reason: "done" }
}
```

### 关键机制

- **流式优先**：`callModel()` 返回 AsyncGenerator，StreamingToolExecutor 在流式过程中并行执行工具
- **模型降级**：Fallback 时已收集的 assistantMessages 标记为 tombstone 并清空，重试整个流式请求
- **上下文压缩**：每轮迭代前评估 token 阈值，超出时触发 autocompact，摘要替换原始消息

## 工具系统

### 工具接口（`src/Tool.ts`）

```typescript
interface Tool<Input, Output, Progress> {
  name: string
  description: string
  validateInput(input: unknown): Input
  canUseTool(ctx: ToolUseContext): boolean
  checkPermissions(input: Input, ctx: ToolPermissionContext): PermissionResult
  call(input: Input): AsyncGenerator<ToolResult<Output, Progress>>
}
```

核心方法链：`validateInput() → canUseTool() → checkPermissions() → call() → ToolResult`

### 工具分类（59+ 工具）

| 分类           | 工具                                                                 |
| -------------- | -------------------------------------------------------------------- |
| **文件操作**   | FileEditTool, FileReadTool, FileWriteTool, GlobTool, GrepTool        |
| **Shell/执行** | BashTool, PowerShellTool, REPLTool                                   |
| **Agent 系统** | AgentTool, TaskCreateTool, TaskUpdateTool, TaskListTool, TaskGetTool |
| **规划**       | EnterPlanModeTool, ExitPlanModeV2Tool, VerifyPlanExecutionTool       |
| **Web/MCP**    | WebFetchTool, WebSearchTool, MCPTool, McpAuthTool                    |
| **调度**       | CronCreateTool, CronDeleteTool, CronListTool                         |
| **其他**       | LSPTool, ConfigTool, SkillTool, EnterWorktreeTool, ExitWorktreeTool  |

### 权限系统

权限规则从 5 个来源汇聚（优先级从高到低）：
1. **session** — 当前会话
2. **project** — 项目 `.claude/settings.json`
3. **user** — 用户级 `~/.claude/settings.json`
4. **managed** — 管理员策略
5. **default** — 默认规则

支持匹配方式：工具名、命令模式（glob）、路径前缀等。

## API 通信层

### 7 种 Provider

| Provider       | 启用方式                      | 说明                                |
| -------------- | ----------------------------- | ----------------------------------- |
| **firstParty** | 默认                          | Anthropic 直连                      |
| **bedrock**    | `ANTHROPIC_BEDROCK_BASE_URL`  | AWS Bedrock                         |
| **vertex**     | `ANTHROPIC_VERTEX_PROJECT_ID` | Google Vertex                       |
| **foundry**    | `ANTHROPIC_CODE_USE_FOUNDRY`  | Anthropic Foundry                   |
| **openai**     | `CLAUDE_CODE_USE_OPENAI=1`    | OpenAI 兼容（Ollama/DeepSeek/vLLM） |
| **gemini**     | `CLAUDE_CODE_USE_GEMINI=1`    | Google Gemini                       |
| **grok**       | `CLAUDE_CODE_USE_GROK=1`      | xAI Grok                            |

优先级：`modelType` 参数 > 环境变量 > 默认 `firstParty`

### 兼容层设计

所有兼容层采用**流适配器模式**：将第三方 API 格式转为 Anthropic 内部格式，下游代码完全不改。

## Feature Flag 系统

```typescript
import { feature } from 'bun:bundle'

// feature() 只能直接用在 if 语句或三元表达式条件位置
if (feature('DAEMON')) { /* ... */ }
const result = feature('BRIDGE_MODE') ? a : b
```

### Build 默认启用（19 个）

- 基础：`BUDDY`, `TRANSCRIPT_CLASSIFIER`, `BRIDGE_MODE`, `AGENT_TRIGGERS_REMOTE`, `CHICAGO_MCP`, `VOICE_MODE`
- 统计/缓存：`SHOT_STATS`, `PROMPT_CACHE_BREAK_DETECTION`, `TOKEN_BUDGET`
- P0 本地：`AGENT_TRIGGERS`, `ULTRATHINK`, `BUILTIN_EXPLORE_PLAN_AGENTS`, `LODESTONE`
- P1 API：`EXTRACT_MEMORIES`, `VERIFICATION_AGENT`, `KAIROS_BRIEF`, `AWAY_SUMMARY`, `ULTRAPLAN`
- P2：`DAEMON`

启用方式：环境变量 `FEATURE_<FLAG_NAME>=1`

## 状态管理

| 文件                         | 职责                                                                 |
| ---------------------------- | -------------------------------------------------------------------- |
| `src/state/AppState.tsx`     | 中心状态类型和 Context Provider（messages、tools、permissions、MCP） |
| `src/state/AppStateStore.ts` | 默认状态和 Store 工厂                                                |
| `src/state/store.ts`         | Zustand 风格 Store（`createStore`）                                  |
| `src/state/selectors.ts`     | 状态选择器                                                           |
| `src/bootstrap/state.ts`     | 模块级单例（session ID、CWD、project root、token counts）            |

## Monorepo 工作区

| 包                                  | 说明                                                     |
| ----------------------------------- | -------------------------------------------------------- |
| `packages/@ant/ink/`                | Forked Ink 框架（components、hooks、keybindings、theme） |
| `packages/@ant/computer-use-mcp/`   | Computer Use MCP server                                  |
| `packages/@ant/computer-use-input/` | 键鼠模拟                                                 |
| `packages/@ant/model-provider/`     | Model provider 抽象层                                    |
| `packages/builtin-tools/`           | 59 个内置工具实现                                        |
| `packages/agent-tools/`             | Agent 工具集                                             |
| `packages/acp-link/`                | ACP 代理服务器（WebSocket → ACP 桥接）                   |
| `packages/mcp-client/`              | MCP 客户端库                                             |
| `packages/remote-control-server/`   | 自托管 RCS（Docker + Web UI）                            |
| `packages/audio-capture-napi/`      | 原生音频捕获                                             |

## 四个核心设计原则

1. **流式优先 (Streaming-first)** — 所有 API 通信都是流式的，StreamingToolExecutor 不等流结束就开始并行执行工具
2. **工具即能力 (Tool as Capability)** — 每个工具是 `Tool<I,O,P>` 结构化类型，通过 `buildTool()` 工厂创建
3. **权限即边界 (Permission as Boundary)** — 双重检查 `validateInput() → checkPermissions()`，5 级权限来源
4. **上下文即记忆 (Context as Memory)** — System Prompt 动态组装，auto-compact 跨压缩边界累计 token 预算

## 参考文档

- `docs/introduction/architecture-overview.mdx` — 五层架构详解（源码级数据流分析）
- `docs/diagrams/agent-loop.mmd` — 完整 Agentic Loop 流程图
- `docs/diagrams/agent-loop-simple.mmd` — 简化版流程图
- `docs/features/` — 各功能模块详细说明
- `docs/internals/feature-flags.mdx` — Feature flag 系统详解