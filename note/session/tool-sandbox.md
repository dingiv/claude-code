# Tool Calling 沙箱与权限机制
claude code 是采用权限审查加沙箱隔离来提高安全性的。

```
Claude 决定调用工具
        │
        ▼
canUseTool(tool, input, context, msg, toolUseID)
  [useCanUseTool hook]
        │
        ▼
hasPermissionsToUseTool()
        │
        ├─ Step 1a: DENY 规则？           → deny
        ├─ Step 1b: ASK 规则？            → ask
        ├─ Step 1c: checkPermissions()    → allow/deny/ask/passthrough
        ├─ Step 1d: 工具拒绝？            → deny
        ├─ Step 1e: 需要用户交互？        → ask
        ├─ Step 1f: 内容 ASK 规则？       → ask
        ├─ Step 1g: 安全检查？            → ask
        ├─ Step 2a: Bypass 模式？         → allow
        ├─ Step 2b: ALLOW 规则？          → allow
        ├─ Step 3: passthrough → ask
        │
        ▼
  模式变换
        │
        ├─ dontAsk → ask 转 deny
        ├─ auto    → AI 分类器判断
        └─ headless → hooks + deny
        │
        ▼
  返回结果
        │
        ├─ allow → 执行工具
        ├─ deny  → 返回拒绝给 Claude
        └─ ask   → 显示权限 UI
              │
              ├─ 用户接受 → allow（可选持久化规则）
              └─ 用户拒绝 → deny
```

## 权限模式

`src/types/permissions.ts` 定义了 6 种权限模式：

| 模式                | 外部可见 | 行为                                         |
| ------------------- | -------- | -------------------------------------------- |
| `default`           | 是       | 大部分工具操作需用户确认                     |
| `plan`              | 是       | 计划模式 — 限制工具访问，Claude 只规划不执行 |
| `acceptEdits`       | 是       | 自动接受文件编辑和部分 shell 命令            |
| `bypassPermissions` | 是       | 跳过所有权限提示（危险）                     |
| `dontAsk`           | 是       | 静默拒绝所有需要提示的工具，不显示 UI        |
| `auto`              | 是       | AI 分类器自动判断允许/拒绝，无需人工干预     |

内部模式：`bubble`（仅内部使用）。

**模式切换**（Shift+Tab）：
```
default → acceptEdits → plan → auto → bypassPermissions → default
```

**模式初始化优先级**：
1. `--dangerously-skip-permissions` CLI 参数 → `bypassPermissions`
2. `--permission-mode <mode>` CLI 参数
3. `settings.json` 中的 `permissions.defaultMode`
4. 默认 `default`

组织级 Statsig gate（`tengu_disable_bypass_permissions_mode`）或设置（`permissions.disableBypassPermissionsMode`）可强制禁用 bypass 模式。

---

## 权限评估流水线

核心函数：`hasPermissionsToUseTool`（`src/utils/permissions/permissions.ts:473`）

基于规则的检查

```
1a. 工具级 DENY 规则？          → deny
1b. 工具级 ASK 规则？           → ask（沙箱自动允许时跳过）
1c. tool.checkPermissions()     → allow / deny / ask / passthrough
    [工具特定的检查，如 Bash 命令解析]
1d. 工具被拒绝？                 → deny
1e. 需要用户交互的工具？         → ask（即使 bypass 模式）
1f. 内容特定的 ASK 规则？        → ask（优先于 bypass）
1g. 安全检查（.git/.claude 等） → ask（bypass 免疫）
```

## 沙箱执行

### 三层架构

| 层级 | 组件 | 职责 |
|------|------|------|
| 应用适配层 | `src/utils/sandbox/sandbox-adapter.ts` | 将 Claude Code 设置转换为沙箱配置，管理生命周期 |
| 沙箱运行时 | `@anthropic-ai/sandbox-runtime@0.0.44` | 平台特定的沙箱构建、代理服务器、域名过滤 |
| 原生 seccomp | `vendor/seccomp-src/apply-seccomp.c` | BPF 过滤器生成与 `prctl(PR_SET_SECCOMP)` 应用 |

执行流程：
```
BashTool.tsx → shouldUseSandbox() → Shell.ts:exec()
  → SandboxManager.wrapWithSandbox()
    → BaseSandboxManager.wrapWithSandbox()        [sandbox-runtime]
      → wrapCommandWithSandboxLinux()             [bwrap 命令构建]
```

### Linux 内核机制

#### 使用的内核特性

| 机制 | bwrap 参数 | 内核特性 | 作用 |
|------|-----------|---------|------|
| 网络隔离 | `--unshare-net` | `CLONE_NEWNET` | 创建空网络栈，无接口无路由 |
| PID 隔离 | `--unshare-pid` | `CLONE_NEWPID` | 沙箱内进程自视为 PID 1，无法 `kill` 宿主进程 |
| 文件系统隔离 | `--ro-bind / /` + `--bind` | `mount()` 命名空间 | 根文件系统只读，仅允许路径可写 |
| 独立 /proc | `--proc /proc` | 新 PID 命名空间挂载 | 隐藏宿主进程信息 |
| 独立 /dev | `--dev /dev` | 设备挂载 | 私有设备节点，隔离宿主设备 |
| 进程会话隔离 | `--new-session` | `setsid()` | 脱离控制终端，防止向父进程发信号 |
| 父进程死亡联动 | `--die-with-parent` | `PR_SET_PDEATHSIG` | 父进程死亡时自动杀死沙箱进程 |
| Unix Socket 封禁 | `apply-seccomp` | `prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER)` | BPF 过滤 `socket(AF_UNIX)` |

**未使用的内核特性**：无 cgroups、无 Landlock、无 AppArmor、无 User Namespace（`--unshare-user`）。

> 不创建 User Namespace 意味着沙箱进程保留调用者的 UID。如果调用者是 root，沙箱内也是 root。这是设计权衡——bwrap 要么需要 root，要么需要内核 `kernel.unprivileged_userns_clone` 支持。

#### Seccomp-BPF 细节

**过滤器生成**（`vendor/seccomp-src/seccomp-unix-block.c`）：
```c
ctx = seccomp_init(SCMP_ACT_ALLOW);  // 默认：允许所有 syscall
seccomp_rule_add(ctx, SCMP_ACT_ERRNO(EPERM), SCMP_SYS(socket), 1,
                 SCMP_A0(SCMP_CMP_MASKED_EQ, 0xffffffff, AF_UNIX));
```

- 默认策略：允许全部 syscall（`SCMP_ACT_ALLOW`）
- 仅拦截：`socket(AF_UNIX, ...)` → 返回 `EPERM`
- 预编译 BPF 存储于 `vendor/seccomp/{x64,arm64}/unix-block.bpf`（x64: 112 字节，arm64: 96 字节）

**过滤器应用**（`vendor/seccomp-src/apply-seccomp.c`）：
```c
prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0);    // 防止提权（seccomp 前置条件）
prctl(PR_SET_SECCOMP, SECCOMP_MODE_FILTER, &prog);  // 加载 BPF
execvp(command_argv[0], command_argv);       // exec 替换为用户命令
```

`PR_SET_NO_NEW_PRIVS` 确保无法通过 exec setuid 程序提权。BPF 过滤器通过 exec 继承给所有子进程，且**不可卸载**。

仅支持 x64 和 arm64。32 位 x86（ia32）不受支持——`socketcall()` 多路复用 syscall 会绕过过滤器。

#### 文件系统命名空间

**基本策略**：根文件系统只读，逐路径开放写权限。

```bash
bwrap \
  --ro-bind / /                              # 整个根文件系统只读
  --bind /home/user/project /home/user/project     # 项目目录可写
  --bind /tmp/claude-xxx /tmp/claude-xxx           # 临时目录可写
  ...
```

**写保护路径**（覆盖在可写路径之上）：
```bash
  --ro-bind .git/hooks .git/hooks              # 禁止修改 git hooks
  --ro-bind .claude/settings.json .claude/settings.json  # 禁止修改 settings
```

**读保护路径**（挂载空 tmpfs 隐藏内容）：
```bash
  --tmpfs /path/to/secret                     # 用空 tmpfs 覆盖，内容完全不可见
```

**不存在的写保护路径**（防止创建）：
```bash
  --ro-bind /dev/null /home/user/.bashrc       # 将 /dev/null 绑到路径上
```
挂载后会在宿主产生 0 字节幽灵文件，每次命令结束后 `cleanupBwrapMountPoints()` 清理。

**符号链接保护**：所有路径在允许写入前通过 `fs.realpathSync()` 解析符号链接，并用 `isSymlinkOutsideBoundary()` 检查是否指向预期边界外。防止通过符号链接逃逸到非允许目录。

**强制写保护列表**：

| 路径 | 原因 |
|------|------|
| `.claude/settings.json`, `.claude/settings.local.json` | 防止沙箱逃逸（修改 settings 可以禁用沙箱） |
| `.claude/skills/`, `.claude/commands/`, `.claude/agents/` | 防止 hook/技能注入 |
| `.git/hooks/`, `.git/config` | 防止 git RCE |
| `.gitconfig`, `.bashrc`, `.zshrc`, `.profile` | 防止 shell 劫持 |
| `.vscode/`, `.idea/` | 防止 IDE 扩展滥用 |
| 裸仓库文件（`HEAD`, `objects/`, `refs/`） | 防止在 cwd 种植裸仓库劫持 git |

#### 网络隔离架构

`--unshare-net` 提供的是**全有或全无**的网络隔离（无网络接口）。但命令通常需要网络访问。解决方案是三层代理桥接：

```
宿主机（bwrap 外部）                        沙箱内部（bwrap 内部）
=====================                       =====================
HTTP 代理 (:random_port)               <-- socat TCP-LISTEN:3128
  ^                                        |
  | socat 桥接                              | socat UNIX-CONNECT
  v                                        v
Unix Socket: /tmp/claude-http-XXX.sock   （bind-mount 进沙箱）

SOCKS5 代理 (:random_port)              <-- socat TCP-LISTEN:1080
  ^                                        |
  | socat 桥接                              | socat UNIX-CONNECT
  v                                        v
Unix Socket: /tmp/claude-socks-XXX.sock  （bind-mount 进沙箱）
```

**初始化流程**：
1. 宿主启动 HTTP CONNECT 代理 + SOCKS5 代理（`127.0.0.1:<random>`），实现域名 allow/deny 过滤
2. 宿主启动 socat 桥接：Unix Socket ↔ TCP（代理端口）
3. 命令执行时，Unix Socket bind-mount 进沙箱
4. 沙箱内启动 socat 监听 TCP 端口，转发到 Unix Socket
5. 设置环境变量：`HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`、`GIT_SSH_COMMAND`

**域名过滤在宿主代理层执行**，不在内核层。不尊重 `HTTP_PROXY`/`ALL_PROXY` 环境变量的工具理论上可以绕过域名限制——但由于 `--unshare-net` 的存在，它们没有原始网络访问能力。

**Seccomp 与网络的交互**：

```bash
# 沙箱内执行的脚本
socat TCP-LISTEN:3128,fork,reuseaddr UNIX-CONNECT:/tmp/... &   # 不受 seccomp 限制
socat TCP-LISTEN:1080,fork,reuseaddr UNIX-CONNECT:/tmp/... &   # 不受 seccomp 限制
trap "kill %1 %2 2>/dev/null; exit" EXIT
apply-seccomp /path/to/unix-block.bpf /bin/bash -c '<user_command>'  # 受 seccomp 限制
```

socat 桥接进程在 seccomp 加载前启动，保留 Unix Socket 能力。只有用户命令被 seccomp 限制。

### 完整的 bwrap 命令结构

```bash
bwrap \
  --new-session \
  --die-with-parent \
  --unshare-net \
  --bind /tmp/claude-http-XXX.sock /tmp/claude-http-XXX.sock \
  --bind /tmp/claude-socks-XXX.sock /tmp/claude-socks-XXX.sock \
  --setenv HTTP_PROXY http://localhost:3128 \
  --setenv HTTPS_PROXY http://localhost:3128 \
  --setenv ALL_PROXY socks5h://localhost:1080 \
  --setenv NO_PROXY localhost,127.0.0.1 \
  --setenv GIT_SSH_COMMAND 'ssh -o ProxyCommand=socat - PROXY:localhost:%h:%p,proxyport=3128' \
  --ro-bind / / \
  --bind /home/user/project /home/user/project \
  --bind /tmp/claude-xxx /tmp/claude-xxx \
  --ro-bind /home/user/project/.git/hooks /home/user/project/.git/hooks \
  --ro-bind /home/user/project/.claude/settings.json /home/user/project/.claude/settings.json \
  --ro-bind /dev/null /home/user/.bashrc \
  --dev /dev \
  --unshare-pid \
  --proc /proc \
  -- /bin/bash -c '
    socat TCP-LISTEN:3128,fork,reuseaddr UNIX-CONNECT:/tmp/... >/dev/null 2>&1 &
    socat TCP-LISTEN:1080,fork,reuseaddr UNIX-CONNECT:/tmp/... >/dev/null 2>&1 &
    trap "kill %1 %2 2>/dev/null; exit" EXIT
    apply-seccomp /path/to/unix-block.bpf /bin/bash -c "<user_command>"
  '
```

### 安全性评估

#### 能防御的威胁

| 威胁 | 防御机制 | 内核级保证 |
|------|---------|-----------|
| 写入非允许路径 | mount 命名空间（只读 bind mount） | 是 |
| 读取敏感目录 | tmpfs 覆盖 + /dev/null 绑定 | 是 |
| 直接网络访问 | `CLONE_NEWNET` 空网络栈 | 是 |
| 域名级别的网络限制 | 宿主代理过滤 | 否（应用层） |
| 信号攻击宿主进程 | PID 命名空间隔离 | 是 |
| 窥探宿主 /proc | 新 PID 命名空间 + 独立 /proc | 是 |
| 符号链接逃逸 | 路径解析 + 边界检查 | 否（应用层） |
| Unix Socket 创建 | seccomp-BPF | 是 |
| Git hooks RCE | .git/hooks 写保护 | 是 |
| Settings 修改逃逸 | settings 文件写保护 | 是 |
| Shell 配置劫持 | .bashrc/.zshrc 写保护 | 是 |
| 裸仓库 git 劫持 | HEAD/objects/refs 写保护 + 清理 | 是 |
| 沙箱进程孤儿化 | `PR_SET_PDEATHSIG` | 是 |
| 控制终端信号 | `setsid()` | 是 |
| 提权（setuid） | `PR_SET_NO_NEW_PRIVS` | 是 |

#### 已知局限（源码中明确承认）

| 局限 | 原因 | 风险等级 |
|------|------|---------|
| 继承的 Unix Socket FD | seccomp 仅拦截 `socket()` 系统调用，不拦截 `bind()`/`connect()`/`sendto()` 在已打开的 FD 上操作 | 低 |
| SCM_RIGHTS FD 传递 | Unix Socket 可以通过 `sendmsg(SCM_RIGHTS)` 传递 FD，seccomp 未拦截 | 低 |
| 32 位 x86 绕过 | ia32 的 `socketcall()` 多路复用 syscall 绕过 BPF 过滤器 | 中（不受支持的平台） |
| 代理级信任 | 不尊重 `HTTP_PROXY` 的工具可绕过域名限制（但仍无原始网络） | 低 |
| 无 User Namespace | 保留调用者 UID，root 运行时沙箱内也是 root | 中 |
| 幽灵文件竞态 | 写保护路径不存在时挂载产生 0 字节文件，清理前短暂存在 | 低 |
| Glob 模式时序 | `*.env` 等规则在配置时展开为具体路径，之后新建的文件不被覆盖 | 低 |
| 无 cgroups 限制 | 沙箱进程无 CPU/内存/IO 资源限制，可 fork bomb 或 OOM | 中 |

### 沙箱配置与 bwrap 参数映射

| 配置字段 | bwrap 参数 | 说明 |
|---------|-----------|------|
| `filesystem.allowWrite[]` | `--bind <path> <path>` | 可写目录 |
| `filesystem.denyWrite[]` | `--ro-bind <path> <path>` | 覆盖写权限为只读 |
| `filesystem.denyRead[]` | `--tmpfs <dir>` 或 `--ro-bind /dev/null <file>` | 隐藏内容 |
| `filesystem.allowRead[]` | `--ro-bind <path> <path>`（在 tmpfs 下重新绑定） | 在读保护中开洞 |
| `network.allowedDomains` | `--unshare-net` + 代理过滤 | 网络隔离 + 域名白名单 |
| `network.allowAllUnixSockets: false` | `apply-seccomp <bpf> <cmd>` | 禁止创建 Unix Socket |
| 无写配置时 | `--bind / /`（整个根可写，无隔离） | 退化为无文件系统保护 |

### shouldUseSandbox 决策门控

`packages/builtin-tools/src/tools/BashTool/shouldUseSandbox.ts`

全部满足才启用沙箱：
1. `SandboxManager.isSandboxingEnabled()` 返回 true（settings 启用 + 平台支持 + 依赖就绪）
2. `dangerouslyDisableSandbox` 未设置
3. 命令不匹配 `excludedCommands` 模式

复合命令拆分检查：`&&`、`||`、`;`、`|` 分隔的每个子命令独立检查，防止 `allowed_cmd && malicious_cmd` 绕过。
