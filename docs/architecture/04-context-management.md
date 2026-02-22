# OpenClaw Agent 上下文管理

## 概述

OpenClaw 的上下文管理系统负责维护 Agent 与用户之间的对话上下文，包括消息历史、上下文窗口管理、自动压缩（Compaction）以及长期记忆系统。上下文管理的核心挑战是在有限的 LLM 上下文窗口内保持对话的连贯性和相关性。

## 1. 上下文窗口管理

### 1.1 上下文窗口信息 (`src/agents/context-window-guard.ts`)

每个 LLM 模型有不同的上下文窗口大小，系统通过多种方式获取：

```typescript
// 常量定义
const CONTEXT_WINDOW_HARD_MIN_TOKENS = 4096;    // 硬性最小值
const CONTEXT_WINDOW_WARN_BELOW_TOKENS = 16384; // 警告阈值
```

`resolveContextWindowInfo()` 解析当前模型的上下文窗口参数：

1. 优先使用配置中的自定义值
2. 其次使用模型目录中的发现值
3. 默认回退到 `DEFAULT_CONTEXT_TOKENS`

### 1.2 上下文窗口守卫

`evaluateContextWindowGuard()` 评估是否需要触发上下文压缩：

```
当前 token 使用量 → 与上下文窗口比较
    → 超过阈值 → 触发压缩
    → 低于最小值 → 发出警告
    → 正常范围 → 继续运行
```

### 1.3 模型上下文缓存 (`src/agents/context.ts`)

`MODEL_CACHE` 全局缓存各模型的上下文窗口大小：

```typescript
// 从模型发现中填充
function applyDiscoveredContextWindows(params: {
  cache: Map<string, number>;
  models: ModelEntry[];
}) {
  // 当多个供应商提供同一模型时，选取较小的窗口（安全偏好）
}

// 从用户配置中填充（优先级更高）
function applyConfiguredContextWindows(params: {
  cache: Map<string, number>;
  modelsConfig: ModelsConfig | undefined;
})
```

## 2. 会话消息历史

### 2.1 消息格式

会话历史中的消息基于 `pi-agent-core` 的 `AgentMessage` 类型：

```typescript
type AgentMessage = {
  role: "user" | "assistant";
  content: ContentBlock[];
  // 可能包含 tool_use, tool_result 等块
};
```

### 2.2 历史限制 (`src/agents/pi-embedded-runner/history.ts`)

系统支持对历史消息进行截断：

```typescript
// DM (私聊) 历史限制
function getDmHistoryLimitFromSessionKey(params: {
  sessionKey?: string;
  config?: OpenClawConfig;
}): number | undefined

// 通用历史限制
function getHistoryLimitFromSessionKey(params: {
  sessionKey?: string;
  config?: OpenClawConfig;
}): number | undefined

// 执行截断
function limitHistoryTurns(
  messages: AgentMessage[],
  limit: number
): AgentMessage[]
```

支持在配置中设置：
- 全局历史轮次限制
- 每通道历史限制
- DM 专属历史限制

### 2.3 会话历史清理 (`src/agents/pi-embedded-runner/run.ts`)

每次运行前会对会话历史进行清理：

1. **工具结果截断**: `truncateOversizedToolResultsInSession()` — 超大工具结果被截断
2. **Token 估算**: 检查当前历史的 token 总量
3. **Google Turn 修复**: `applyGoogleTurnOrderingFix()` — 修复 Google 模型的消息顺序要求
4. **消息对修复**: `repairToolUseResultPairing()` — 确保 tool_use/tool_result 成对

## 3. 上下文压缩 (Compaction)

### 3.1 压缩概述 (`src/agents/compaction.ts`)

当会话上下文接近窗口限制时，系统自动触发压缩：

```typescript
const BASE_CHUNK_RATIO = 0.4;   // 基础分块比例
const MIN_CHUNK_RATIO = 0.15;   // 最小分块比例
const SAFETY_MARGIN = 1.2;      // 20% 安全缓冲（补偿 estimateTokens 的不精确）
```

### 3.2 压缩策略

```
会话消息 → estimateMessagesTokens() → 超过阈值?
    → 是 → splitMessagesByTokenShare() 分块
        → 每块调用 generateSummary() 生成摘要
        → mergeSummaries() 合并摘要
        → 替换原始消息为压缩摘要
    → 否 → 保持不变
```

### 3.3 Token 估算

```typescript
function estimateMessagesTokens(messages: AgentMessage[]): number {
  // 安全处理: 工具结果的 details 字段可能包含不可信的大量数据
  const safe = stripToolResultDetails(messages);
  return safe.reduce((sum, message) => sum + estimateTokens(message), 0);
}
```

### 3.4 消息分块

```typescript
function splitMessagesByTokenShare(
  messages: AgentMessage[],
  parts: number = 2
): AgentMessage[][] {
  // 按 token 比例将消息分成大致相等的块
  // 避免在单条消息中间切割
}
```

### 3.5 摘要生成

每个分块调用 LLM 生成摘要，然后合并：

```
分块1 → generateSummary() → 摘要1 ─┐
分块2 → generateSummary() → 摘要2 ──┼→ mergeSummaries() → 最终摘要
分块N → generateSummary() → 摘要N ─┘
```

合并指令: "Merge these partial summaries into a single cohesive summary. Preserve decisions, TODOs, open questions, and any constraints."

### 3.6 压缩安全

- 工具结果的 `details` 字段在压缩前被剥离（防止不可信数据进入 LLM）
- 压缩有重试机制 (`retryAsync`)
- 压缩超时保护 (`compaction-safety-timeout`)

## 4. 记忆系统

### 4.1 核心记忆模块 (`extensions/memory-core/`)

提供基于文件的长期记忆存储：

- 记忆以 Markdown 文件形式存储
- 支持 `MEMORY.md` 主记忆文件和 `memory/*.md` 辅助文件
- 记忆搜索通过 `memory_search` 工具
- 记忆获取通过 `memory_get` 工具

### 4.2 向量记忆 (`extensions/memory-lancedb/`)

基于 LanceDB 的向量检索记忆系统：

- 将记忆片段嵌入为向量
- 支持语义搜索
- 适用于大量记忆数据的高效检索

### 4.3 记忆工具 (`src/agents/memory-search.ts`)

Agent 可通过内置工具访问记忆系统：

```
## Memory Recall (系统提示词中)
Before answering anything about prior work, decisions, dates, people,
preferences, or todos: run memory_search on MEMORY.md + memory/*.md;
then use memory_get to pull only the needed lines.
```

### 4.4 记忆引用模式 (`MemoryCitationsMode`)

```typescript
type MemoryCitationsMode = "off" | "on";
```

- **on**: 回复中包含 `Source: <path#line>` 引用
- **off**: 不包含文件路径和行号

## 5. 引导上下文 (Bootstrap Context)

### 5.1 引导文件 (`src/agents/bootstrap-files.ts`)

Agent 启动时加载初始上下文文件：

- `AGENTS.md` / `CLAUDE.md`: 项目级指南
- 工作区特定的引导文件
- Agent 配置中指定的额外文件

### 5.2 上下文文件 (`EmbeddedContextFile`)

```typescript
type EmbeddedContextFile = {
  path: string;
  content: string;
};
```

引导文件被注入到系统提示词中，为 Agent 提供项目背景知识。

## 6. 系统提示词上下文参数

### 6.1 提示词参数 (`src/agents/system-prompt-params.ts`)

系统提示词中嵌入动态上下文：

- 当前时间
- 可用工具列表
- 技能列表
- 记忆指令
- 用户身份信息
- 工作区信息
- 沙箱环境信息

### 6.2 上下文注入流程

```
Agent 运行 → buildSystemPrompt()
    → 身份行
    → Skills 部分 (按需加载)
    → Memory Recall 部分
    → 用户身份部分
    → 工具说明部分
    → 工作区上下文
    → 引导文件内容
    → 当前时间
```

## 7. 会话转录管理

### 7.1 转录策略 (`src/agents/transcript-policy.ts`)

控制会话转录的保留和清理策略：

- 转录大小限制
- 自动清理过期转录
- 安全保护（防止敏感信息泄露）

### 7.2 会话文件修复 (`src/agents/session-file-repair.ts`)

处理损坏的会话文件：

- 检测并修复不完整的 JSON
- 修复工具使用/结果配对不一致
- 清理无效的消息格式

### 7.3 转录修复 (`src/agents/session-transcript-repair.ts`)

```typescript
function repairToolUseResultPairing(messages: AgentMessage[]): AgentMessage[]
function stripToolResultDetails(messages: AgentMessage[]): AgentMessage[]
```

## 8. 共享状态 (`src/shared/`)

### 8.1 全局状态

- 全局变量 (`src/globals.ts`): 日志级别等运行时标志
- 配置管理 (`src/config/config.ts`): 全局配置对象

### 8.2 配置会话设置 (`src/config/`)

会话相关配置：

```typescript
// 在 config 中
session: {
  dmScope: "main" | "per-peer" | "per-channel-peer" | "per-account-channel-peer";
  identityLinks: Record<string, string[]>;
  historyLimit?: number;
  dmHistoryLimit?: number;
}
```

## 9. 上下文流转图

```
┌─────────────────────────────────────────────────────┐
│                 上下文管理架构                        │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌───────────────┐                                  │
│  │ Bootstrap     │  启动时加载                       │
│  │ Context       │──────────┐                       │
│  └───────────────┘          │                       │
│                             ▼                       │
│  ┌───────────────┐   ┌──────────────┐              │
│  │ System Prompt │──→│ 运行时上下文  │              │
│  │ (动态构建)    │   │ (每次运行)    │              │
│  └───────────────┘   └──────┬───────┘              │
│                             │                       │
│  ┌───────────────┐          ▼                       │
│  │ 会话历史      │   ┌──────────────┐              │
│  │ (持久化)      │──→│ LLM 请求     │              │
│  └───────┬───────┘   │ 上下文窗口    │              │
│          │           └──────┬───────┘              │
│          │                  │                       │
│  ┌───────▼───────┐   ┌─────▼────────┐             │
│  │ 压缩/截断     │   │ 记忆系统      │             │
│  │ (自动触发)    │   │ (按需检索)    │             │
│  └───────────────┘   └──────────────┘              │
│                                                     │
└─────────────────────────────────────────────────────┘
```

## 关键源码文件索引

| 文件路径 | 说明 |
|---------|------|
| `src/agents/context.ts` | 模型上下文窗口缓存 |
| `src/agents/context-window-guard.ts` | 上下文窗口守卫 |
| `src/agents/compaction.ts` | 上下文压缩引擎 |
| `src/agents/defaults.ts` | 默认值 (模型、token 数等) |
| `src/agents/pi-embedded-runner/history.ts` | 历史限制与截断 |
| `src/agents/bootstrap-files.ts` | 引导上下文加载 |
| `src/agents/system-prompt.ts` | 系统提示词构建 |
| `src/agents/system-prompt-params.ts` | 提示词参数 |
| `src/agents/memory-search.ts` | 记忆搜索工具 |
| `src/agents/session-file-repair.ts` | 会话文件修复 |
| `src/agents/session-transcript-repair.ts` | 转录修复 |
| `src/agents/transcript-policy.ts` | 转录策略 |
| `extensions/memory-core/` | 核心记忆扩展 |
| `extensions/memory-lancedb/` | 向量记忆扩展 |
| `src/config/config.ts` | 配置管理 |
