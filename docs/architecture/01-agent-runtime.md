# OpenClaw Agent 运行时机制

## 概述

OpenClaw 的 Agent 运行时是整个系统的核心引擎，负责管理 AI Agent 的启动、运行、消息处理和生命周期。Agent 基于 `@mariozechner/pi-coding-agent` 库构建，采用嵌入式运行模式 (Embedded Pi Agent)，支持多种 LLM 供应商。

## 1. 启动流程

### 1.1 CLI 入口链

Agent 的启动从 CLI 入口开始，经过多层初始化：

```
openclaw.mjs → src/entry.ts → src/cli/run-main.js → src/index.ts → src/cli/program.ts
```

**`src/entry.ts`** — 最外层入口点：
- 设置进程标题 `process.title = "openclaw"`
- 安装进程警告过滤器 (`installProcessWarningFilter`)
- 标准化环境变量 (`normalizeEnv`)
- 处理 `--no-color` 参数
- 抑制 Node.js ExperimentalWarning（通过子进程重新生成机制）
- 解析 CLI Profile 参数
- 最终调用 `src/cli/run-main.js` 的 `runCli()`

**`src/index.ts`** — 核心模块入口：
- 加载 `.env` 文件 (`loadDotEnv`)
- 标准化环境变量
- 确保 CLI 可执行文件在 PATH 中 (`ensureOpenClawCliOnPath`)
- 启用控制台输出捕获 (`enableConsoleCapture`)
- 断言运行时版本满足要求 (`assertSupportedRuntime`，Node 22+)
- 构建 Commander 程序 (`buildProgram`)
- 安装全局异常处理器

### 1.2 运行时环境 (`src/runtime.ts`)

`RuntimeEnv` 类型定义了运行时 I/O 接口：

```typescript
type RuntimeEnv = {
  log: (...args: unknown[]) => void;   // 日志输出
  error: (...args: unknown[]) => void; // 错误输出
  exit: (code: number) => void;        // 退出进程
};
```

- `defaultRuntime`: 生产运行时，`exit` 会恢复终端状态后调用 `process.exit`
- `createNonExitingRuntime()`: 测试用运行时，`exit` 抛出异常而非终止进程
- I/O 函数会清除活跃的进度行 (`clearActiveProgressLine`) 后再输出

### 1.3 子进程重生成机制 (`entry.ts`)

当 Node.js 未配置 `--disable-warning=ExperimentalWarning` 时：

1. 设置守卫变量 `OPENCLAW_NODE_OPTIONS_READY=1` 防止递归
2. 通过 `child_process.spawn` 使用相同参数重新启动自身
3. 通过 `attachChildProcessBridge` 桥接子进程信号
4. 父进程等待子进程退出后中继退出码

## 2. Agent 核心引擎

### 2.1 嵌入式 Pi Agent 运行器

Agent 的核心运行逻辑位于 `src/agents/pi-embedded-runner/` 目录：

| 文件 | 职责 |
|------|------|
| `run.ts` | Agent 运行主入口 (`runEmbeddedPiAgent`) |
| `runs.ts` | 运行状态管理 (活跃运行追踪、消息队列、中止) |
| `run/attempt.ts` | 单次 LLM 调用尝试 (`runEmbeddedAttempt`) |
| `run/payloads.ts` | 构建嵌入式运行的请求负载 |
| `run/params.ts` | 运行参数类型定义 |
| `compact.ts` | 会话压缩（上下文过长时自动摘要） |
| `history.ts` | 会话历史限制与 DM 历史截断 |
| `lanes.ts` | 并发通道 (Lane) 管理 |
| `model.ts` | 模型解析 |
| `types.ts` | 核心类型定义 |

### 2.2 Agent 运行流程

```
消息到达 → resolveAgentRoute() → resolveSessionKey()
    → runEmbeddedPiAgent()
        → enqueueSession → enqueueGlobal (双层队列)
            → resolveRunWorkspaceDir()   // 解析工作目录
            → resolveModel()             // 解析模型+供应商
            → getApiKeyForModel()        // 获取 API 密钥
            → resolveContextWindowInfo() // 上下文窗口守卫
            → buildEmbeddedRunPayloads() // 构建 LLM 请求
            → runEmbeddedAttempt()       // 执行 LLM 调用
                ← 流式响应处理
                ← 工具调用执行
                ← 错误分类 → 故障转移/重试
            → compactIfNeeded()          // 上下文过长时压缩
```

**双层排队**: 每次运行通过 Session Lane（保证同一会话串行）和 Global Lane（全局并发控制）双重排队。

### 2.3 关键运行参数 (`RunEmbeddedPiAgentParams`)

每次 Agent 运行需要以下参数：

- **sessionKey**: 唯一标识当前会话的键
- **model**: 要使用的 LLM 模型标识
- **provider**: LLM 供应商 (anthropic, openai, google 等)
- **contextWindow**: 上下文窗口大小 (token 数)
- **systemPrompt**: 系统提示词
- **tools**: 可用工具集合
- **messages**: 会话消息历史

### 2.4 Token 使用量跟踪

系统通过 `UsageAccumulator` 精确跟踪每次运行的 token 消耗：

```typescript
type UsageAccumulator = {
  input: number;         // 累计输入 token
  output: number;        // 累计输出 token
  cacheRead: number;     // 累计缓存读取
  cacheWrite: number;    // 累计缓存写入
  total: number;         // 累计总量
  lastCacheRead: number; // 最近一次 API 调用的缓存读取（非累计）
  lastCacheWrite: number;
  lastInput: number;
};
```

**关键设计**: 使用最近一次 API 调用的缓存字段（而非累计）计算上下文大小，因为多轮工具调用中累计的 `cacheRead` 会 N 倍膨胀。

### 2.5 运行状态管理 (`runs.ts`)

系统通过全局 `ACTIVE_EMBEDDED_RUNS` Map 追踪活跃运行：

```typescript
type EmbeddedPiQueueHandle = {
  queueMessage: (text: string) => Promise<void>;
  isStreaming: () => boolean;
  isCompacting: () => boolean;
  abort: () => void;
};

const ACTIVE_EMBEDDED_RUNS = new Map<string, EmbeddedPiQueueHandle>();
```

**消息排队条件**: `queueEmbeddedPiMessage()` 仅在活跃运行存在、正在流式传输、且未在压缩时接受消息。

**等待机制**: `waitForEmbeddedPiRunEnd()` 支持超时等待（默认 15 秒），使用 waiter 集合通知所有等待者。

### 2.6 安全处理

- **Anthropic 拒绝魔法字符串**: 自动将 `ANTHROPIC_MAGIC_STRING_TRIGGER_REFUSAL` 替换为脱敏版本，防止会话污染
- **工具结果截断**: `truncateOversizedToolResultsInSession()` 在运行前截断超大工具结果

## 3. 多 Agent 系统

### 3.1 Agent 作用域 (`src/agents/agent-scope.ts`)

OpenClaw 支持多 Agent 配置：

- 通过 `config.agents.list` 定义多个 Agent
- 每个 Agent 有独立的 ID、模型配置、工具权限
- `resolveDefaultAgentId()`: 解析默认 Agent ID

### 3.2 子 Agent (Subagent) 系统

位于 `src/agents/subagent-*.ts`：

- **subagent-registry.ts**: 子 Agent 注册表，管理子 Agent 生命周期
- **subagent-spawn.ts**: 子 Agent 生成逻辑
- **subagent-depth.ts**: 子 Agent 嵌套深度限制
- **subagent-announce.ts**: 子 Agent 结果通知

子 Agent 支持：
- 独立的会话上下文
- 深度限制防止无限递归
- 结果通过通知队列返回给父 Agent

### 3.3 Agent 工作区 (`src/agents/workspace.ts`)

每个 Agent 维护独立的工作区：

- `resolveAgentWorkspaceDir()`: 解析 Agent 工作区目录
- 工作区包含会话数据、配置文件、引导文件
- 支持工作区模板 (`workspace-templates.ts`)
- 引导文件系统 (`bootstrap-files.ts`) 在 Agent 启动时加载初始上下文

## 4. 认证与模型管理

### 4.1 认证配置 (`src/agents/model-auth.ts`)

- 支持多认证配置 (Auth Profiles)
- 自动轮换 API 密钥
- 冷却期管理 (失败后自动暂停使用)
- 支持 Anthropic、OpenAI、Google、HuggingFace 等供应商

### 4.2 模型目录 (`src/agents/model-catalog.ts`)

- 维护可用模型列表
- 支持模型别名和兼容性映射
- 自动发现供应商提供的模型
- 上下文窗口信息缓存

### 4.3 故障转移 (`src/agents/model-fallback.ts`)

当 LLM 调用失败时的多层故障转移机制：

**错误分类** (`FailoverReason`):
- 认证错误 (`auth`) — API 密钥无效
- 计费错误 (`billing`) — 账户余额不足
- 限流错误 (`rate-limit`) — 请求频率过高
- 超时错误 (`timeout`) — 请求超时
- 上下文溢出 (`context-overflow`) — 输入超过模型限制
- 图片尺寸错误 (`image-size`, `image-dimension`)
- 压缩失败 (`compaction-failure`)

**故障转移候选收集** (`ModelCandidate`):
```typescript
type ModelCandidate = { provider: string; model: string };
// 通过 createModelCandidateCollector() 去重收集
// 按配置的允许列表过滤
// 支持模型别名解析 (buildModelAliasIndex)
```

**故障转移流程**:
1. 主模型失败 → 分类错误原因
2. AbortError（非超时）→ 直接重抛，不进入故障转移
3. 按优先级尝试 `config.agents.defaults.model.fallbacks` 中的备用模型
4. 标记失败的认证配置进入冷却期 (`markAuthProfileFailure`)
5. 所有候选耗尽 → 汇总所有尝试的错误摘要
6. 支持思考级别降级 (`pickFallbackThinkingLevel`)

**图片生成故障转移**: 独立的候选解析（`resolveImageFallbackCandidates`），支持 `agents.defaults.imageModel.fallbacks`。

## 5. 工具系统

### 5.1 工具定义 (`src/agents/pi-tools.ts`)

Agent 拥有丰富的工具集：

- **Bash 工具** (`bash-tools.ts`): 命令执行 (PTY/非 PTY 模式)
- **通道工具** (`channel-tools.ts`): 消息发送/接收
- **内存工具** (`memory-search.ts`): 记忆搜索/检索
- **OpenClaw 工具** (`openclaw-tools.ts`): 会话管理/子 Agent

### 5.2 工具执行管线

```
LLM 请求工具调用 → 工具策略检查 (tool-policy.ts)
    → 前置钩子 (before-tool-call)
    → 工具执行
    → 后置钩子 (after-tool-call)
    → 结果返回给 LLM
```

### 5.3 沙箱执行 (`src/agents/sandbox/`)

敏感工具（如 Bash）在沙箱中执行：

- Docker/Podman 容器隔离
- 挂载路径限制
- 命令审批机制 (`exec-approval-request.ts`)

## 6. 系统提示词构建

`src/agents/system-prompt.ts` 负责构建发送给 LLM 的系统提示词：

### 6.1 提示词模式

- **full**: 完整提示词 (主 Agent 使用)
- **minimal**: 精简提示词 (子 Agent 使用)
- **none**: 仅基础身份行

### 6.2 提示词组成部分

1. 身份行 (Agent 名称与角色)
2. 技能部分 (可用 Skills)
3. 记忆部分 (Memory Recall 指令)
4. 授权发送者 (用户身份)
5. 工具说明 (可用工具的使用指南)
6. 工作区与运行时上下文
7. 当前时间信息

## 7. 进程与并发管理

### 7.1 Lane 并发系统 (`src/agents/pi-embedded-runner/lanes.ts`)

Agent 运行使用双层 Lane 排队：

- **Session Lane** (`resolveSessionLane`): 保证同一会话的请求串行处理
- **Global Lane** (`resolveGlobalLane`): 全局并发控制，防止过载

每次 `runEmbeddedPiAgent()` 调用通过 `enqueueSession(() => enqueueGlobal(async () => ...))` 嵌套排队。

### 7.2 命令队列 (`src/process/command-queue.ts`)

`enqueueCommandInLane()` 实现 Lane 级 FIFO 排队：

- 基于 Lane 名称的隔离队列
- 全局队列大小监控
- 支持异步等待队列完成

### 7.3 进程执行 (`src/process/exec.ts`)

- `runExec()`: 带超时的命令执行
- `runCommandWithTimeout()`: 限时命令执行
- 子进程桥接 (`child-process-bridge.ts`): 信号转发（SIGINT, SIGTERM）

## 8. 关键数据流

```
┌──────────────┐    ┌───────────────┐    ┌──────────────┐
│   Channel    │───→│  Gateway      │───→│  Routing     │
│  (Feishu,    │    │  (server.ts)  │    │  (resolve-   │
│   Discord..) │    │               │    │   route.ts)  │
└──────────────┘    └───────────────┘    └──────┬───────┘
                                                │
                         ┌──────────────────────┘
                         ▼
                   ┌──────────────┐    ┌──────────────┐
                   │  Session     │───→│  Pi Embedded  │
                   │  Management  │    │  Runner       │
                   └──────────────┘    └──────┬───────┘
                                              │
                         ┌────────────────────┘
                         ▼
                   ┌──────────────┐    ┌──────────────┐
                   │  LLM Provider│←──→│  Tool System  │
                   │  (Anthropic, │    │  (bash, send, │
                   │   OpenAI...) │    │   memory...)  │
                   └──────────────┘    └──────────────┘
```

## 关键源码文件索引

| 文件路径 | 说明 |
|---------|------|
| `src/entry.ts` | CLI 入口 |
| `src/runtime.ts` | 运行时环境定义 |
| `src/index.ts` | 核心模块入口与导出 |
| `src/agents/pi-embedded-runner/run.ts` | Agent 运行主逻辑 |
| `src/agents/pi-embedded-runner/runs.ts` | 运行状态管理 |
| `src/agents/pi-embedded-subscribe.ts` | 流式响应订阅 |
| `src/agents/system-prompt.ts` | 系统提示词构建 |
| `src/agents/agent-scope.ts` | Agent 作用域管理 |
| `src/agents/model-auth.ts` | 模型认证管理 |
| `src/agents/model-fallback.ts` | 故障转移机制 |
| `src/agents/compaction.ts` | 上下文压缩 |
| `src/agents/subagent-registry.ts` | 子 Agent 注册表 |
| `src/process/command-queue.ts` | 命令队列 |
