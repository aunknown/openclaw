# OpenClaw Agent 上下文管理

## 概述

OpenClaw 的上下文管理系统负责维护 Agent 与用户之间的对话上下文，包括消息历史、上下文窗口管理、自动压缩（Compaction）以及长期记忆系统。上下文管理的核心挑战是在有限的 LLM 上下文窗口内保持对话的连贯性和相关性。

## 1. 上下文窗口管理

### 1.1 上下文窗口信息 (`src/agents/context-window-guard.ts`)

每个 LLM 模型有不同的上下文窗口大小，系统通过多种方式获取：

```typescript
const CONTEXT_WINDOW_HARD_MIN_TOKENS = 16_000;  // 硬性最小值 (低于此则阻止运行)
const CONTEXT_WINDOW_WARN_BELOW_TOKENS = 32_000; // 警告阈值

type ContextWindowSource = "model" | "modelsConfig" | "agentContextTokens" | "default";
```

`resolveContextWindowInfo()` 解析当前模型的上下文窗口参数，优先级从高到低：

1. **modelsConfig**: 用户在 `config.models.providers[provider].models` 中指定的 `contextWindow`
2. **model**: 模型自身报告的上下文窗口（模型发现值）
3. **default**: `DEFAULT_CONTEXT_TOKENS` 回退默认值
4. **agentContextTokens 上限**: 若 `config.agents.defaults.contextTokens` 存在且更小，则覆盖

### 1.2 上下文窗口守卫

`evaluateContextWindowGuard()` 评估当前上下文窗口是否安全：

```typescript
type ContextWindowGuardResult = ContextWindowInfo & {
  shouldWarn: boolean;   // tokens > 0 && tokens < warnBelow (32K)
  shouldBlock: boolean;  // tokens > 0 && tokens < hardMin (16K)
};
```

- **shouldBlock=true**: 上下文窗口过小，阻止 Agent 运行
- **shouldWarn=true**: 发出警告但继续运行
- **正常**: 无需干预

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
const DEFAULT_PARTS = 2;         // 默认分块数
```

### 3.2 自适应分块比例

当消息平均大小较大时，动态减小分块比例：

```typescript
function computeAdaptiveChunkRatio(messages: AgentMessage[], contextWindow: number): number {
  const avgRatio = (avgTokens * SAFETY_MARGIN) / contextWindow;
  // 平均消息 > 10% 上下文窗口时减小比例
  if (avgRatio > 0.1) {
    return Math.max(MIN_CHUNK_RATIO, BASE_CHUNK_RATIO - avgRatio * 2);
  }
  return BASE_CHUNK_RATIO;
}
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

**按 Token 比例分块** (`splitMessagesByTokenShare`)：
```typescript
// 按 token 比例将消息分成大致相等的块
// 在消息边界切割，从不在单条消息中间切割
function splitMessagesByTokenShare(messages: AgentMessage[], parts = 2): AgentMessage[][]
```

**按最大 Token 分块** (`chunkMessagesByMaxTokens`)：
```typescript
// 按固定 token 上限分块，超大单条消息独立成块
function chunkMessagesByMaxTokens(messages: AgentMessage[], maxTokens: number): AgentMessage[][]
```

### 3.5 多阶段摘要生成 (`summarizeInStages`)

完整的压缩流程支持渐进式回退：

```
消息列表 → 数量/token 够大? → splitMessagesByTokenShare() 分块
    → 每块 → summarizeWithFallback()
        → 尝试完整摘要 → 成功 → 部分摘要
        → 失败 → 回退 1: 跳过超大消息 (>50% 上下文窗口)，仅摘要小消息
        → 失败 → 回退 2: 生成统计描述 "Context contained N messages (M oversized)"
    → 所有部分摘要 → mergeSummaries() → 最终摘要
```

**合并指令**: "Merge these partial summaries into a single cohesive summary. Preserve decisions, TODOs, open questions, and any constraints."

**重试机制**：每次 `generateSummary()` 调用使用 `retryAsync`，最多 3 次尝试，指数退避 500ms-5000ms，20% 抖动。

### 3.6 历史修剪 (`pruneHistoryForContextShare`)

当历史超过上下文预算时，渐进式丢弃最早的消息：

```typescript
function pruneHistoryForContextShare(params: {
  messages: AgentMessage[];
  maxContextTokens: number;
  maxHistoryShare?: number;  // 默认 0.5 (历史最多占上下文的 50%)
}): {
  messages: AgentMessage[];
  droppedMessagesList: AgentMessage[];  // 用于生成摘要
  droppedChunks: number;
  droppedTokens: number;
  keptTokens: number;
  budgetTokens: number;
}
```

丢弃后自动修复 tool_use/tool_result 配对 (`repairToolUseResultPairing`)，删除孤立的 tool_result。

### 3.7 压缩安全

- 工具结果的 `details` 字段在压缩前被剥离（防止不可信数据进入 LLM）
- `isOversizedForSummary()`: 单条消息超过上下文 50% 时跳过
- 压缩超时保护 (`compaction-safety-timeout`)
- AbortError 不重试

## 4. 记忆系统

### 4.1 记忆搜索配置 (`src/agents/memory-search.ts`)

记忆系统通过 `ResolvedMemorySearchConfig` 配置，支持深度定制：

```typescript
type ResolvedMemorySearchConfig = {
  enabled: boolean;
  sources: Array<"memory" | "sessions">;       // 搜索源
  provider: "openai" | "local" | "gemini" | "voyage" | "auto";  // 嵌入供应商
  fallback: "openai" | "gemini" | "local" | "voyage" | "none";  // 回退供应商
  store: {
    driver: "sqlite";
    path: string;                               // 默认: {stateDir}/memory/{agentId}.sqlite
    vector: { enabled: boolean; extensionPath?: string };
  };
  chunking: { tokens: number; overlap: number };  // 默认: 400 tokens, 80 overlap
  query: {
    maxResults: number;       // 默认: 6
    minScore: number;         // 默认: 0.35
    hybrid: {                 // 混合搜索配置
      enabled: boolean;       // 默认: true
      vectorWeight: number;   // 默认: 0.7
      textWeight: number;     // 默认: 0.3
      candidateMultiplier: number;  // 默认: 4
      mmr: { enabled: boolean; lambda: number };       // 最大边际相关性
      temporalDecay: { enabled: boolean; halfLifeDays: number };  // 时间衰减
    };
  };
  sync: {
    onSessionStart: boolean;  // 会话开始时同步
    onSearch: boolean;        // 搜索前同步
    watch: boolean;           // 文件监听
    watchDebounceMs: number;  // 默认: 1500ms
    sessions: { deltaBytes: number; deltaMessages: number };  // 默认: 100KB/50条
  };
};
```

**嵌入模型默认值**：

| 供应商 | 默认模型 |
|--------|---------|
| OpenAI | `text-embedding-3-small` |
| Gemini | `gemini-embedding-001` |
| Voyage | `voyage-4-large` |

### 4.2 核心记忆模块 (`extensions/memory-core/`)

提供基于文件的长期记忆存储：

- 记忆以 Markdown 文件形式存储
- 支持 `MEMORY.md` 主记忆文件和 `memory/*.md` 辅助文件
- 记忆搜索通过 `memory_search` 工具
- 记忆获取通过 `memory_get` 工具

### 4.3 向量记忆 (`extensions/memory-lancedb/`)

基于 LanceDB 的向量检索记忆系统：

- 将记忆片段嵌入为向量
- 支持语义搜索
- 适用于大量记忆数据的高效检索

### 4.4 SQLite 存储 (`memory-search.ts`)

记忆存储使用 SQLite 作为驱动：

- 路径: `{stateDir}/memory/{agentId}.sqlite`，支持 `{agentId}` 模板变量
- 支持 SQLite 向量扩展 (`vector.extensionPath`)
- 嵌入缓存减少重复计算

### 4.5 混合搜索系统

默认启用混合搜索（向量 + 关键词）：

```
查询 → 向量搜索 (权重 0.7) + 关键词搜索 (权重 0.3)
    → 合并候选 (candidateMultiplier=4, 即搜索 maxResults×4 个候选)
    → [可选] MMR 去重 (lambda=0.7)
    → [可选] 时间衰减 (halfLife=30天)
    → 返回 top maxResults (默认 6)，过滤 minScore < 0.35
```

### 4.6 记忆工具 (系统提示词中)

```
## Memory Recall
Before answering anything about prior work, decisions, dates, people,
preferences, or todos: run memory_search on MEMORY.md + memory/*.md;
then use memory_get to pull only the needed lines.
```

### 4.7 记忆引用模式 (`MemoryCitationsMode`)

```typescript
type MemoryCitationsMode = "off" | "on";
```

- **on**: 回复中包含 `Source: <path#line>` 引用
- **off**: 不包含文件路径和行号

## 5. 引导上下文 (Bootstrap Context)

### 5.1 引导文件 (`src/agents/bootstrap-files.ts`)

Agent 启动时加载初始上下文文件，经过两阶段处理：

```typescript
async function resolveBootstrapContextForRun(params: {
  workspaceDir: string;
  config?: OpenClawConfig;
  sessionKey?: string;
  agentId?: string;
}): Promise<{
  bootstrapFiles: WorkspaceBootstrapFile[];
  contextFiles: EmbeddedContextFile[];
}>
```

**加载流程**：
1. `loadWorkspaceBootstrapFiles()` — 扫描工作区发现引导文件
2. `filterBootstrapFilesForSession()` — 按会话键过滤
3. `applyBootstrapHookOverrides()` — 应用钩子覆盖
4. `buildBootstrapContextFiles()` — 构建上下文文件（受大小限制）

**文件类型**: `AGENTS.md` / `CLAUDE.md`（项目级指南）、工作区特定引导文件、Agent 配置指定的额外文件

**大小限制**: 通过 `resolveBootstrapMaxChars()` 和 `resolveBootstrapTotalMaxChars()` 控制单文件和总量上限。

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

根据 LLM 供应商动态调整转录处理策略：

```typescript
type TranscriptPolicy = {
  sanitizeMode: "full" | "images-only";    // 内容清理模式
  sanitizeToolCallIds: boolean;             // 工具调用 ID 卫生化
  toolCallIdMode?: "strict" | "strict9";   // ID 格式 (Mistral 用 strict9)
  repairToolUseResultPairing: boolean;      // 修复工具配对
  preserveSignatures: boolean;              // 保留签名
  sanitizeThinkingSignatures: boolean;      // 清理思考签名
  dropThinkingBlocks: boolean;              // 丢弃思考块 (GitHub Copilot Claude)
  applyGoogleTurnOrdering: boolean;         // Google 消息顺序修复
  validateGeminiTurns: boolean;             // Gemini 轮次验证
  validateAnthropicTurns: boolean;          // Anthropic 轮次验证
  allowSyntheticToolResults: boolean;       // 允许合成工具结果
};
```

**供应商特定行为**：

| 供应商 | 清理模式 | 工具 ID 卫生化 | 配对修复 | 轮次排序 |
|--------|---------|---------------|---------|---------|
| Anthropic | full | strict | yes | no |
| Google/Gemini | full | strict | yes | yes |
| Mistral | full | strict9 | no | no |
| OpenAI | images-only | no | no | no |

### 7.2 会话文件修复 (`src/agents/session-file-repair.ts`)

处理损坏的会话文件：

- 检测并修复不完整的 JSON
- 修复工具使用/结果配对不一致
- 清理无效的消息格式

### 7.3 转录修复 (`src/agents/session-transcript-repair.ts`)

```typescript
function repairToolUseResultPairing(messages: AgentMessage[]): {
  messages: AgentMessage[];
  droppedOrphanCount: number;   // 被丢弃的孤立 tool_result 数量
}
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
