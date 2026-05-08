# 新文件夹中打开 Claude Code 的初始化流程

## 一、会话生命周期视角

整个启动过程分为 9 个阶段。

### Phase 0: 模块求值（`main()` 之前）

- `bootstrap/state.ts` 模块加载时同步创建全局单例 `STATE`
- **`sessionId = randomUUID()`** — 会话 ID 在进程启动时立即生成，而非用户交互时
- `originalCwd` 通过 `realpathSync()` 解析符号链接并做 NFC 标准化
- `projectRoot` 初始设为 CWD，后续可能被 git root 覆盖

### Phase 1-2: 快速路径检查 → Commander 注册

- `cli.tsx` 的 `main()` 排除 `--version`、`--daemon` 等快速路径后，开始捕获键盘输入（防止启动期间击键丢失），然后导入 `main.tsx`
- 检测交互模式（`isNonInteractive`）、客户端类型、提前加载 settings

### Phase 3-4: `init()` — 一次性基础设施引导

整个启动过程中最重的一步，按顺序执行：

| 步骤 | 作用 |
|------|------|
| `enableConfigs()` | 读取 `settings.json` 配置 |
| `applySafeConfigEnvironmentVariables()` | 应用安全的环境变量 |
| `detectCurrentRepository()` | Git 仓库检测 |
| `initUser()` | 预热用户邮箱缓存 |
| `preconnectAnthropicApi()` | 预热 TCP+TLS 握手（~100-200ms） |
| `configureGlobalAgents()` | 代理配置 |

### Phase 5: `setup()` — 会话级初始化

- `setCwd(cwd)` 正式设定工作目录
- **`initSessionMemory()`** — 初始化会话记忆系统（后台任务）
- `initializeFileChangedWatcher()` — 启动文件变更监听
- 插件 hooks 加载、技能初始化、遥测 sinks 初始化

### Phase 6: Trust Dialog — 新文件夹的关键分水岭

**新文件夹与已有文件夹的最大区别**：

```
checkHasTrustDialogAccepted() → 遍历 ~/.claude.json 的 projects map
```

- 新文件夹没有对应的 project 配置 → 返回 `false`
- **弹出信任对话框**："Is this a project you created or one you trust?"
- 用户接受后：
  - 如果是 home 目录 → 仅内存标记，不持久化
  - **否则 → `saveCurrentProjectConfig({ hasTrustDialogAccepted: true })` 写入 `~/.claude.json`**

信任确认后的连锁反应：

- GrowthBook 重新初始化（拿到认证 headers）
- **`getSystemContext()` 预取** — 现在可以安全执行 git 命令了
- MCP server 审批、CLAUDE.md 外部引用审批
- **`applyConfigEnvironmentVariables()`** — 应用全部环境变量（含危险变量）
- 遥测初始化

### Phase 7: REPL 启动

渲染 `<App><REPL /></App>`，启动延迟预取任务，用户看到输入提示符。

---

## 二、记忆系统视角

### CLAUDE.md 发现机制

`getMemoryFiles()`（`claudemd.ts:790`）从多个层级向上搜索：

```
/etc/claude-code/CLAUDE.md          ← 系统级策略（通常不存在）
~/.claude/CLAUDE.md                 ← 全局用户指令
~/.claude/rules/*.md                ← 全局用户规则
CWD/CLAUDE.md                       ← 项目级（新文件夹：不存在）
CWD/.claude/CLAUDE.md               ← 项目级（新文件夹：不存在）
CWD/.claude/rules/*.md              ← 项目规则（新文件夹：不存在）
CWD/CLAUDE.local.md                 ← 本地覆盖（新文件夹：不存在）
→ 向上遍历到根目录...
memory/MEMORY.md                    ← AutoMem（首次运行：不存在）
```

新文件夹的结果：只有 `~/.claude/CLAUDE.md` 和 `~/.claude/rules/*.md` 会被发现（如果用户创建过的话）。项目级 CLAUDE.md 全部 ENOENT 静默跳过。

### 会话记忆（`initSessionMemory()`）

在 `setup()` 中作为后台任务启动，负责加载会话历史相关的记忆状态。

### Git 上下文

`getSystemContext()` 对非 git 文件夹返回 `null` — 没有分支信息、没有 recent commits、没有 git status。

---

## 三、总结：新文件夹 vs 已有项目文件夹

| 维度 | 新文件夹 | 已有项目 |
|------|---------|---------|
| Trust Dialog | **必须弹出**（无持久化记录） | 跳过（`~/.claude.json` 已有记录） |
| CLAUDE.md | 仅全局级（`~/.claude/`） | 全局 + 项目级（多层叠加） |
| Git 上下文 | `null`（无 git 仓库） | 完整 status/log/branch |
| MEMORY.md | 不存在 | 如果有则自动加载到系统提示 |
| Session ID | 每次随机 UUID | 可通过 `--continue` 恢复历史会话 |
| 项目配置 | 首次创建（trust 记录写入 `~/.claude.json`） | 已存在 |

## 四、核心设计思想

1. **信任是准入门槛** — 只有用户明确信任了一个目录，Claude Code 才会执行 git 命令、应用危险环境变量、初始化遥测
2. **记忆分层叠加** — 从系统级到全局级到项目级逐层发现，新文件夹自然退化为仅全局记忆
3. **会话 ID 即时生成** — 在模块求值阶段就已创建，不依赖任何用户交互或 I/O
4. **Trust 持久化位置** — 写入 `~/.claude.json` 的 `projects` map（key 为项目路径），而非项目本地文件，确保不可被项目本身篡改
