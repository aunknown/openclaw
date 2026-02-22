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
        → resolveModel()           // 解析使用的模型
        → buildEmbeddedRunPayloads() // 构建 LLM 请求
        → runEmbeddedAttempt()      // 执行 LLM 调用
            ← 流式响应处理
            ← 工具调用执行
            ← 错误重试/故障转移
        → compactIfNeeded()         // 上下文过长时压缩
```

### 2.3 关键运行参数 (`RunEmbeddedPiAgentParams`)

每次 Agent 运行需要以下参数：

- **sessionKey**: 唯一标识当前会话的键
- **model**: 要使用的 LLM 模型标识
- **provider**: LLM 供应商 (anthropic, openai, google 等)
- **contextWindow**: 上下文窗口大小 (token 数)
- **systemPrompt**: 系统提示词
- **tools**: 可用工具集合
- **messages**: 会话消息历史

### 2.4 运行状态管理 (`runs.ts`)

系统维护全局的活跃运行追踪：

- `queueEmbeddedPiMessage()`: 将新消息排入运行队列
- `isEmbeddedPiRunActive()`: 检查指定会话是否有活跃运行
- `isEmbeddedPiRunStreaming()`: 检查是否正在流式传输
- `abortEmbeddedPiRun()`: 中止指定会话的运行
- `waitForEmbeddedPiRunEnd()`: 等待运行完成

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

当 LLM 调用失败时的故障转移机制：

- 自动检测错误类型 (认证、计费、限流、超时)
- 按优先级尝试备用认证配置
- 标记失败的配置并记录冷却期
- 支持手动指定降级思考级别

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

## 7. 进程管理

### 7.1 命令队列 (`src/process/command-queue.ts`)

所有 Agent 运行通过命令队列调度：

- Lane (通道) 级别的并发控制
- 队列大小监控
- 支持优先级排队

### 7.2 进程执行 (`src/process/exec.ts`)

- `runExec()`: 带超时的命令执行
- `runCommandWithTimeout()`: 限时命令执行
- 子进程桥接 (`child-process-bridge.ts`): 信号转发

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
