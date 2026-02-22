# Agent 会话上下文与过期机制深度分析

## 1. 概述

OpenClaw 的 agent 每次交互时，会从磁盘上的 JSONL 会话文件中加载**完整的历史消息**，然后通过多层处理管线对其进行裁剪、清洗和压缩，最终构造出发送给 LLM API 的上下文。会话过期采用**双模式**策略（每日重置 + 空闲超时），由配置驱动。

---

## 2. 每次交互携带多少历史会话消息

### 2.1 基线：加载全部历史消息

每次 agent 交互时，核心入口 `runEmbeddedAttempt()`（`src/agents/pi-embedded-runner/run/attempt.ts:225`）会：

1. 打开会话 JSONL 文件：`SessionManager.open(params.sessionFile)`
2. 通过 `createAgentSession()` 创建 agent 会话，此时 `session.messages` 包含**该会话文件中的全部历史消息**

也就是说，**默认情况下，所有历史消息都会被加载到内存中**。

### 2.2 历史消息裁剪管线

加载完整历史后，消息经过以下处理管线（`attempt.ts:711-741`）：

```
原始消息 → sanitizeSessionHistory() → validateGeminiTurns() / validateAnthropicTurns()
         → limitHistoryTurns() → sanitizeToolUseResultPairing() → replaceMessages()
```

#### 第一步：sanitizeSessionHistory（历史清洗）
- 针对 Google/Gemini 模型，清洗不兼容的消息格式
- 移除无效的 thinking block
- 修复 tool_use/tool_result 配对问题

#### 第二步：validateTurns（轮次验证）
- `validateAnthropicTurns()`：确保消息符合 Anthropic API 的轮次交替要求
- `validateGeminiTurns()`：确保消息符合 Gemini API 格式要求

#### 第三步：limitHistoryTurns（历史轮次限制）⭐ 关键裁剪点
**文件**: `src/agents/pi-embedded-runner/history.ts:15-36`

```typescript
export function limitHistoryTurns(
  messages: AgentMessage[],
  limit: number | undefined,
): AgentMessage[] {
  if (!limit || limit <= 0 || messages.length === 0) {
    return messages;
  }
  let userCount = 0;
  let lastUserIndex = messages.length;
  for (let i = messages.length - 1; i >= 0; i--) {
    if (messages[i].role === "user") {
      userCount++;
      if (userCount > limit) {
        return messages.slice(lastUserIndex);
      }
      lastUserIndex = i;
    }
  }
  return messages;
}
```

这个函数**从后向前**计数用户消息（user turns），只保留最近 N 轮用户消息及其关联的 assistant 响应。

**限制值的解析**（`history.ts:43-109`）：

| 会话类型 | 配置路径 | 优先级 |
|---------|---------|--------|
| DM 会话 | `channels.<provider>.dms.<userId>.historyLimit` | 最高（per-DM 覆盖） |
| DM 会话 | `channels.<provider>.dmHistoryLimit` | 次高（provider 级默认） |
| Channel/Group | `channels.<provider>.historyLimit` | provider 级 |
| 其他 | 无 | 不限制（返回 undefined） |

**如果没有配置 `historyLimit`/`dmHistoryLimit`，则不做裁剪，全部历史消息都会传入。**

#### 第四步：sanitizeToolUseResultPairing（工具配对修复）
- 裁剪后可能产生孤立的 tool_result（其对应的 tool_use 被裁掉了），这一步修复这些断裂的配对关系

### 2.3 上下文压缩（Compaction）机制

当上下文超出模型 context window 时，系统会触发 **compaction**（压缩）：

**文件**: `src/agents/compaction.ts`

- **默认 context window**: 200,000 tokens（`src/agents/defaults.ts:6`）
- **触发时机**: SDK 自动检测或 context overflow 错误时触发
- **工作原理**:
  1. 将历史消息分块（`splitMessagesByTokenShare()`），默认分 2 块
  2. 使用 LLM 为旧消息生成摘要（`generateSummary()`）
  3. 用摘要替换旧消息，保留近期消息
  4. 摘要保留：决策、TODO、开放问题、约束条件

**Compaction 的关键参数**:
- `BASE_CHUNK_RATIO = 0.4`：默认取 40% 的历史进行压缩
- `MIN_CHUNK_RATIO = 0.15`：消息过大时最少压缩 15%
- `SAFETY_MARGIN = 1.2`：20% 的 token 估算安全裕量
- `maxHistoryShare = 0.5`：历史消息最多占 context window 的 50%（`pruneHistoryForContextShare()`）

**Overflow 自动压缩**（`run.ts:477-676`）：
- 最多尝试 3 次自动压缩（`MAX_OVERFLOW_COMPACTION_ATTEMPTS = 3`）
- 压缩超时限制 5 分钟（`EMBEDDED_COMPACTION_TIMEOUT_MS = 300_000`）

### 2.4 Context Pruning（上下文裁剪扩展）

**文件**: `src/agents/pi-extensions/context-pruning/settings.ts`

除 compaction 外，还有一个**独立的 context pruning 扩展**，基于 cache TTL 对旧的工具返回结果进行裁剪：

```typescript
export const DEFAULT_CONTEXT_PRUNING_SETTINGS = {
  mode: "cache-ttl",
  ttlMs: 5 * 60 * 1000,           // 5 分钟 TTL
  keepLastAssistants: 3,           // 始终保留最近 3 条 assistant 消息
  softTrimRatio: 0.3,              // 上下文占用 30% 时软裁剪
  hardClearRatio: 0.5,             // 上下文占用 50% 时硬清除
  minPrunableToolChars: 50_000,    // 工具返回 >50KB 才触发裁剪
  softTrim: {
    maxChars: 4_000,               // 软裁剪后最大保留 4KB
    headChars: 1_500,              // 保留开头 1.5KB
    tailChars: 1_500,              // 保留末尾 1.5KB
  },
  hardClear: {
    enabled: true,
    placeholder: "[Old tool result content cleared]",
  },
};
```

这意味着：超过 5 分钟的旧工具返回结果，如果占用上下文超过 30%，会被截断为头尾各 1.5KB；超过 50% 时直接替换为占位符。

### 2.5 Group Chat 历史限制

**文件**: `src/auto-reply/reply/history.ts`

群聊场景有额外的历史限制机制：

```typescript
export const DEFAULT_GROUP_HISTORY_LIMIT = 50;   // 默认保留最近 50 条群聊消息
export const MAX_HISTORY_KEYS = 1000;             // 最多追踪 1000 个群聊 key（LRU 淘汰）
```

当群聊历史 key 超过 1000 个时，采用 LRU 策略淘汰最旧的记录。

### 2.6 Tool Result 截断

**文件**: `src/agents/pi-embedded-runner/tool-result-truncation.ts`

过大的工具返回结果会被截断，以避免占用过多 context 空间。

### 2.7 小结：每次交互实际携带多少消息

| 场景 | 携带的历史消息量 |
|------|----------------|
| 新会话/首次交互 | 0 条（空白上下文） |
| 短会话，无 historyLimit | 全部历史消息 |
| 配置了 historyLimit=N | 最近 N 轮 user turn 及关联响应 |
| 上下文接近溢出 | SDK 自动触发 compaction，用摘要替代旧消息 |
| 上下文已溢出 | 最多 3 次自动 compaction 重试 |
| 群聊会话 | 额外限制最近 50 条群消息（DEFAULT_GROUP_HISTORY_LIMIT） |
| 旧 tool result（>5min） | context pruning 自动截断或清除 |

---

## 3. 会话过期机制

### 3.1 会话重置策略（Session Reset Policy）

**文件**: `src/config/sessions/reset.ts`

会话过期有两种模式，可同时生效：

#### 模式 A：每日定时重置（Daily Reset）

```typescript
export const DEFAULT_RESET_MODE: SessionResetMode = "daily";
export const DEFAULT_RESET_AT_HOUR = 4;  // 凌晨 4 点
```

- **默认行为**：每天凌晨 4:00 后首次交互时，如果 `updatedAt` 早于今天凌晨 4:00，会话被判定为过期
- **判定逻辑**（`evaluateSessionFreshness()`, `reset.ts:139-159`）：
  ```typescript
  const staleDaily = dailyResetAt != null && params.updatedAt < dailyResetAt;
  ```
- **可配置**：通过 `session.reset.atHour` 设置重置小时（0-23）

#### 模式 B：空闲超时（Idle Timeout）

```typescript
export const DEFAULT_IDLE_MINUTES = 60;  // 默认 60 分钟
```

- **判定逻辑**：
  ```typescript
  const idleExpiresAt = params.updatedAt + params.policy.idleMinutes * 60_000;
  const staleIdle = idleExpiresAt != null && params.now > idleExpiresAt;
  ```
- 即：如果当前时间超过 `上次更新时间 + idleMinutes 分钟`，会话过期
- **可配置**：通过 `session.reset.idleMinutes` 或 `session.idleMinutes`（旧版）设置

#### 两种模式的组合

```typescript
fresh: !(staleDaily || staleIdle)
```

**只要任一条件触发，会话即被判定为 "不新鲜"（stale），系统会创建新会话。**

### 3.2 分类型重置覆盖

**文件**: `reset.ts:34-50, 84-120`

支持按会话类型设置不同的重置策略：

| 类型 | 判定依据 | 配置键 |
|------|---------|--------|
| `thread` | session key 包含 `:thread:` 或 `:topic:` | `session.resetByType.thread` |
| `group` | session key 包含 `:group:` 或 `:channel:` | `session.resetByType.group` |
| `direct` | 其他（DM 会话） | `session.resetByType.direct`（或旧版 `dm`） |

还支持**按 channel 覆盖**（`resetByChannel`），例如为 Telegram 和 WhatsApp 设置不同策略。

### 3.3 手动重置触发器

**文件**: `src/auto-reply/reply/session.ts:120-205`

用户可以通过发送特定命令手动重置会话：

```typescript
export const DEFAULT_RESET_TRIGGERS = ["/new", "/reset"];
```

- 发送 `/new` 或 `/reset` 立即创建新会话
- 支持带参数：`/new 你好` 会重置会话并以"你好"作为首条消息
- 可通过 `session.resetTriggers` 自定义触发词

### 3.4 会话过期时的处理流程

**文件**: `src/auto-reply/reply/session.ts:232-262`

```
用户消息到达 → initSessionState()
  ├─ 检查是否为重置命令 → 是 → 创建新 sessionId，isNewSession=true
  ├─ 检查 evaluateSessionFreshness()
  │   ├─ fresh=true → 复用现有 session（保留 sessionId、systemSent 等状态）
  │   └─ fresh=false → 创建新 sessionId，isNewSession=true
  └─ 返回 SessionInitResult
```

当 `isNewSession=true` 时：
- 生成新的 `sessionId`（UUID）
- `systemSent` 重置为 false
- `abortedLastRun` 重置为 false
- 如果是手动重置，用户偏好设置（thinking、verbose、reasoning 等）会被保留

### 3.5 Session Store 维护机制

**文件**: `src/config/sessions/store.ts:248-419`

Session Store（sessions.json）有独立的维护机制：

| 维护操作 | 默认值 | 作用 |
|---------|--------|------|
| `pruneStaleEntries` | 30 天 | 删除 30 天未更新的会话条目 |
| `capEntryCount` | 500 | 最多保留 500 个会话条目（按 updatedAt 排序） |
| `rotateSessionFile` | 10 MB | sessions.json 超过 10MB 时轮转 |

维护模式有两种：
- `warn`（默认）：仅警告，不自动删除
- `enforce`：自动执行清理

### 3.6 Cron 会话清理

**文件**: `src/cron/session-reaper.ts`

Cron 任务的运行会话有独立的清理机制：
- 默认保留期：24 小时
- 每 5 分钟检查一次
- 仅清理 `cron:run:` 类型的临时会话

### 3.7 Cache TTL（缓存 TTL）

#### Session Manager Cache
**文件**: `src/agents/pi-embedded-runner/session-manager-cache.ts`
- TTL: 45 秒（`DEFAULT_SESSION_MANAGER_TTL_MS = 45_000`）
- 环境变量: `OPENCLAW_SESSION_MANAGER_CACHE_TTL_MS`
- 作用：避免频繁重新读取会话文件

#### Session Store Cache
**文件**: `src/config/sessions/store.ts:40`
- TTL: 45 秒（`DEFAULT_SESSION_STORE_TTL_MS = 45_000`）
- 环境变量: `OPENCLAW_SESSION_CACHE_TTL_MS`
- 作用：缓存 sessions.json 的读取结果

### 3.8 Agent 执行超时

**文件**: `src/agents/timeout.ts`

```typescript
const DEFAULT_AGENT_TIMEOUT_SECONDS = 600;  // 10 分钟
const MAX_SAFE_TIMEOUT_MS = 2_147_000_000;  // ~24.8 天
```

- 单次 agent 交互的默认超时：10 分钟
- 可通过 `agents.defaults.timeoutSeconds` 配置
- 设为 0 表示无超时（使用 MAX_SAFE_TIMEOUT_MS）

---

## 4. 配置示例

```yaml
# openclaw.json / openclaw.yaml
session:
  # 会话重置配置
  reset:
    mode: "daily"        # "daily" 或 "idle"
    atHour: 4            # 每日重置时间（0-23）
    idleMinutes: 60      # 空闲超时分钟数

  # 按会话类型覆盖
  resetByType:
    direct:
      mode: "idle"
      idleMinutes: 30
    group:
      mode: "daily"
      atHour: 6
    thread:
      mode: "idle"
      idleMinutes: 120

  # Session Store 维护
  maintenance:
    mode: "enforce"      # "warn" 或 "enforce"
    pruneAfter: "30d"    # 过期清理时间
    maxEntries: 500      # 最大会话数

channels:
  telegram:
    historyLimit: 20       # Channel 消息保留 20 轮
    dmHistoryLimit: 50     # DM 消息保留 50 轮
    dms:
      "user123":
        historyLimit: 100  # 特定用户保留 100 轮

agents:
  defaults:
    timeoutSeconds: 600    # Agent 执行超时
    contextTokens: 128000  # 限制 context window 大小
```

---

## 5. 架构图

```
用户消息到达
    │
    ▼
┌─────────────────────────┐
│  initSessionState()     │  判断会话是否过期
│  (session.ts)           │  - daily reset check
│                         │  - idle timeout check
│                         │  - /new /reset command
└──────────┬──────────────┘
           │
    ┌──────┴──────┐
    │ 新会话      │  旧会话（fresh）
    │ 创建新 ID   │  复用 sessionId
    └──────┬──────┘
           │
           ▼
┌─────────────────────────┐
│  runEmbeddedAttempt()   │
│  (attempt.ts)           │
│                         │
│  1. SessionManager.open │  ← 从 JSONL 加载全部消息
│  2. createAgentSession  │
│  3. sanitizeHistory     │  ← 清洗不兼容消息
│  4. validateTurns       │  ← 验证 API 格式
│  5. limitHistoryTurns   │  ← 按 historyLimit 裁剪
│  6. repairPairing       │  ← 修复 tool_use 配对
│  7. replaceMessages     │  ← 用处理后的消息替换
└──────────┬──────────────┘
           │
           ▼
┌─────────────────────────┐
│  Agent 执行 (LLM API)   │
│                         │
│  超时: 10min (default)  │
│  Context: 200k tokens   │
│                         │
│  如果 context overflow: │
│  → auto-compaction      │
│    (最多 3 次重试)       │
│  → 生成摘要替代旧消息    │
└─────────────────────────┘
```

---

## 6. 关键源码文件索引

| 文件 | 职责 |
|------|------|
| `src/agents/pi-embedded-runner/history.ts` | 历史消息轮次限制 |
| `src/agents/pi-embedded-runner/run/attempt.ts` | Agent 单次执行入口 |
| `src/agents/pi-embedded-runner/run.ts` | Agent 执行主循环（含 overflow compaction） |
| `src/agents/compaction.ts` | 上下文压缩/摘要生成 |
| `src/agents/pi-embedded-runner/compact.ts` | 嵌入式 compaction 入口 |
| `src/agents/context-window-guard.ts` | Context window 检测与保护 |
| `src/agents/defaults.ts` | 默认 context tokens (200k) |
| `src/agents/timeout.ts` | Agent 执行超时 |
| `src/config/sessions/reset.ts` | 会话过期/重置策略 |
| `src/config/sessions/store.ts` | Session store 持久化与维护 |
| `src/config/sessions/types.ts` | Session 数据结构与默认值 |
| `src/auto-reply/reply/session.ts` | 会话初始化与新鲜度判定 |
| `src/cron/session-reaper.ts` | Cron 会话清理 |
| `src/agents/pi-embedded-runner/session-manager-cache.ts` | Session 文件缓存 |
| `src/agents/pi-extensions/context-pruning/settings.ts` | Context pruning 配置与默认值 |
| `src/auto-reply/reply/history.ts` | 群聊消息历史管理（50条默认，1000 key LRU） |
