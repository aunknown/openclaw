# OpenClaw 会话管理

## 概述

OpenClaw 的会话管理系统负责为每个用户-Agent-通道组合维护独立的会话状态。会话系统是连接消息路由、Agent 运行器和持久化存储的关键中间层。

## 1. 会话键 (Session Key)

### 1.1 会话键结构 (`src/routing/session-key.ts`)

每个会话由唯一的 Session Key 标识。会话键的构建规则：

```typescript
function buildAgentPeerSessionKey(params: {
  agentId: string;
  mainKey: string;
  channel: string;
  accountId?: string | null;
  peerKind: ChatType;      // "direct" | "channel" | "thread"
  peerId: string | null;
  dmScope?: DmScopeMode;
  identityLinks?: Record<string, string[]>;
}): string
```

会话键格式示例：
```
agent-default:main                       # 主会话
agent-default:discord:default:direct:123 # Discord 私聊
agent-vip:feishu:acct1:channel:group456  # Feishu 群组
```

### 1.2 DM 作用域模式

`dmScope` 控制私聊消息的会话隔离粒度：

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| `main` | 所有 DM 共享一个会话 | 单用户/简单部署 |
| `per-peer` | 每个对话方独立会话 | 多用户，跨通道统一 |
| `per-channel-peer` | 通道+对话方独立 | 多用户，通道隔离 |
| `per-account-channel-peer` | 账户+通道+对话方独立 | 多账户，完全隔离 |

### 1.3 身份链接 (Identity Links)

`identityLinks` 允许将多个通道的用户 ID 关联为同一身份：

```json
{
  "session": {
    "identityLinks": {
      "alice": ["discord:user123", "feishu:ou_456", "telegram:789"]
    }
  }
}
```

当 Alice 从任何通道发送消息时，都会映射到同一个会话。

### 1.4 主会话键

每个 Agent 有一个主会话键 (`mainSessionKey`)：

```typescript
function buildAgentMainSessionKey(params: {
  agentId: string;
  mainKey: string;   // 默认 "main"
}): string
```

主会话键用于 Agent 级别的全局操作。

## 2. 会话存储

### 2.1 存储路径

每个 Agent 有独立的会话存储目录：

```
~/.openclaw/state/
├── agents/
│   ├── main/
│   │   └── sessions/
│   │       ├── sessions.json           # 会话状态存储 (JSON)
│   │       ├── <sessionId>.jsonl       # 转录文件 (JSONL)
│   │       └── <sessionId>-topic-<topicId>.jsonl
│   └── sales/
│       └── sessions/
│           ├── sessions.json
│           └── <sessionId>.jsonl
└── auth/
    └── profiles.json
```

### 2.2 SessionEntry 核心数据结构 (`src/config/sessions/types.ts`)

每个会话键映射到一个 `SessionEntry`，包含丰富的状态信息：

```typescript
type SessionEntry = {
  // 身份与生命周期
  sessionId: string;                  // UUID (跨重置稳定)
  updatedAt: number;                  // 最后更新时间戳 (ms)
  sessionFile?: string;               // JSONL 转录文件路径

  // 子 Agent 嵌套
  spawnedBy?: string;                 // 父会话键
  spawnDepth?: number;                // 0=主, 1=子Agent, 2=孙Agent

  // 模型与供应商覆盖
  modelOverride?: string;             // 覆盖默认模型
  providerOverride?: string;          // 覆盖默认供应商
  authProfileOverride?: string;       // 使用的认证配置

  // Token 使用量追踪
  inputTokens?: number;
  outputTokens?: number;
  totalTokens?: number;
  cacheRead?: number;
  cacheWrite?: number;
  contextTokens?: number;             // 当前模型的上下文窗口大小
  compactionCount?: number;           // 压缩次数

  // 投递路由信息
  channel?: string;                   // 主通道
  lastChannel?: string;               // 最后活跃通道
  lastTo?: string;                    // 最后收件人
  lastAccountId?: string;             // 最后账户
  lastThreadId?: string | number;     // 最后线程 ID
  deliveryContext?: DeliveryContext;   // 完整投递信息

  // 群组/频道元数据
  groupId?: string;
  subject?: string;                   // 群组主题
  chatType?: "group" | "channel" | "direct";

  // 会话策略
  sendPolicy?: "allow" | "deny";      // 发送策略
  queueMode?: string;                 // 消息排队行为
  label?: string;                     // 用户标签 (最长 64 字)
  // ... 更多字段
};
```

### 2.3 转录文件格式 (JSONL)

```jsonl
{"type":"session","version":1,"id":"<sessionId>","timestamp":"2024-01-20T10:30:00Z"}
{"type":"message","message":{"role":"user","content":[{"type":"text","text":"hello"}]}}
{"type":"message","message":{"role":"assistant","content":[{"type":"text","text":"Hi!"}]}}
```

### 2.4 存储缓存机制

```typescript
const SESSION_STORE_CACHE = new Map<string, SessionStoreCacheEntry>();
const DEFAULT_SESSION_STORE_TTL_MS = 45_000; // 45 秒缓存

// loadSessionStore() 行为:
// 1. 检查内存缓存 (TTL 未过期 且 文件 mtime 未变)
// 2. 缓存未命中时从磁盘读取 (Windows 支持 3 次重试)
// 3. 自动迁移遗留字段名
// 4. 深拷贝后缓存
```

## 3. 会话生命周期

### 3.1 会话创建

```
消息到达 → resolveAgentRoute() → 生成 sessionKey
    → 检查会话是否存在
        → 不存在: 创建新会话
            → 初始化 transcript.json
            → 加载引导上下文
        → 存在: 加载现有会话
```

### 3.2 会话活跃状态

会话在以下情况被视为"活跃"：

- 有正在执行的 Agent 运行 (`isEmbeddedPiRunActive`)
- 有正在流式传输的响应 (`isEmbeddedPiRunStreaming`)
- 有排队等待的消息

### 3.3 会话写入锁 (`src/agents/session-write-lock.ts`)

基于文件的分布式锁机制防止并发写入：

```typescript
async function acquireSessionWriteLock(params: {
  sessionFile: string;
  timeoutMs?: number;        // 默认: 10s
  staleMs?: number;          // 默认: 30min (视为死锁)
  maxHoldMs?: number;        // 默认: 5min (看门狗释放)
}): Promise<{ release: () => Promise<void> }>
```

**锁队列系统**: 每个 storePath 维护一个 FIFO 队列序列化写入。

**原子写入**: 使用临时文件 + rename 模式：
- Unix: `.{pid}.{uuid}.tmp` → chmod 0o600 → rename
- Windows: 写入 + 最多 5 次重试 rename

**安全保障**:
- 进程退出时自动释放所有锁 (`process.on("exit")`)
- 看门狗每 60s 检查并释放持有 >5min 的锁
- 过期锁检测 (进程已死或持有时间 >30min)

### 3.4 会话重置策略 (`src/config/sessions/reset.ts`)

支持多种会话重置模式：

| 模式 | 规则 | 默认应用 |
|------|------|---------|
| **daily** | 每天指定时刻重置 (默认 4:00 AM) | 私聊、群组 |
| **idle** | 空闲超过指定分钟数重置 | 线程 (默认 60 min) |

```typescript
function evaluateSessionFreshness(params: {
  updatedAt: number;
  now: number;
  policy: SessionResetPolicy;
}): { fresh: boolean; dailyResetAt?: number; idleExpiresAt?: number }
```

可按通道/类型自定义：
```json
{
  "session": {
    "reset": { "mode": "daily", "atHour": 4 },
    "resetByType": {
      "direct": { "mode": "idle", "idleMinutes": 30 },
      "thread": { "mode": "idle", "idleMinutes": 120 }
    },
    "resetByChannel": {
      "slack": { "mode": "idle", "idleMinutes": 60 }
    }
  }
}
```

### 3.5 存储维护 (自动清理)

`saveSessionStore()` 自动执行维护：

| 操作 | 条件 | 默认值 |
|------|------|-------|
| **修剪** | `updatedAt < now - pruneAfterMs` | 30 天 |
| **限数** | 保留最近 N 条 | 500 条 |
| **轮转** | 文件超过指定大小 | 10MB → 重命名为 `.bak.{timestamp}`，保留 3 个备份 |

## 4. 会话级别配置

### 4.1 会话级模型覆盖 (`src/sessions/model-overrides.ts`)

允许按会话覆盖使用的 LLM 模型：

```typescript
// 配置示例
session: {
  modelOverrides: {
    "agent-default:discord:*": "claude-3-opus"
  }
}
```

### 4.2 会话级发送策略 (`src/sessions/send-policy.ts`)

控制 Agent 回复消息的发送行为：

```typescript
// 发送策略测试验证
// 控制消息是否发送、延迟发送、聚合发送等
```

### 4.3 会话级别覆盖 (`src/sessions/level-overrides.ts`)

支持按会话键模式覆盖日志级别等运行时参数。

### 4.4 会话标签 (`src/sessions/session-label.ts`)

为会话附加人类可读的标签。

## 5. 会话转录事件 (`src/sessions/transcript-events.ts`)

会话转录的事件记录系统：

- 消息接收事件
- Agent 运行开始/结束事件
- 工具调用事件
- 压缩事件
- 错误事件

## 6. 多会话管理

### 6.1 Agent 工具中的会话操作 (`src/agents/openclaw-tools.ts`)

Agent 可通过内置工具管理会话：

- **sessions_list**: 列出所有活跃会话
- **sessions_status**: 查看会话状态
- **sessions_send**: 向指定会话发送消息

### 6.2 子 Agent 会话

子 Agent 生成时创建独立的子会话：

```
父 Agent 会话
├── 子 Agent A 会话 (独立的 sessionKey)
│   └── 子 Agent A1 会话 (深度限制)
└── 子 Agent B 会话
```

### 6.3 会话可见性

会话可见性控制哪些会话对外暴露：

- 内部会话（子 Agent 会话）默认不可见
- 通过配置控制会话的公开范围

## 7. 会话与 Gateway 集成

### 7.1 Gateway 会话管理 (`src/gateway/session-utils.ts`)

Gateway 提供 HTTP/WS API 操作会话：

```typescript
// 会话工具函数
function resolveGatewaySessionDir(sessionKey: string): string
function loadGatewaySessionTranscript(sessionKey: string): AgentMessage[]
```

### 7.2 会话路由 (`src/gateway/server-session-key.ts`)

Gateway 级别的会话键解析：

```typescript
function resolveSessionKeyForRun(params: {
  channel: string;
  accountId?: string;
  peer?: RoutePeer;
  config: OpenClawConfig;
}): string
```

### 7.3 向导会话 (`src/gateway/server-wizard-sessions.ts`)

首次使用时的引导向导会话：

- 引导用户完成初始配置
- 通道连接设置
- 认证配置

### 7.4 会话预览 (`src/gateway/session-preview.ts`)

为 API 消费者提供会话预览信息。

### 7.5 会话补丁 (`src/gateway/sessions-patch.ts`)

支持通过 API 部分更新会话配置：

```typescript
// PATCH /sessions/:key
// 更新会话的特定字段
```

## 8. 输入溯源 (`src/sessions/input-provenance.ts`)

跟踪消息的来源信息：

- 哪个通道接收的消息
- 原始消息 ID
- 发送者标识
- 时间戳

## 9. 会话键工具 (`src/sessions/session-key-utils.ts`)

提供会话键的解析和操作工具：

- 解析会话键的各个组成部分
- 比较会话键
- 会话键模式匹配

## 10. 安全与隔离

### 10.1 会话隔离

- 不同会话键的数据完全隔离
- 会话间无共享状态（除显式配置的身份链接）
- 文件系统级别的目录隔离

### 10.2 工具结果保护 (`src/agents/session-tool-result-guard.ts`)

保护会话中存储的工具结果：

```typescript
// 防止工具结果被篡改或超大
// 提供持久化钩子用于拦截和验证
```

### 10.3 会话 Slug (`src/agents/session-slug.ts`)

为会话生成人类可读的随机标识符，用于日志和调试：

```
"melodic-azure-dragon"
"swift-golden-phoenix"
```

## 11. 生命周期图

```
┌─────────────────────────────────────────────────────────────┐
│                     会话生命周期                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ① 创建                                                     │
│  消息到达 → 路由 → sessionKey 生成 → 初始化会话文件           │
│                                                             │
│  ② 活跃                                                     │
│  消息入队 → Agent 运行 → 流式响应 → 回复派发                  │
│       ↕         ↕           ↕         ↕                     │
│  写入锁获取  工具执行    压缩检查   转录更新                   │
│                                                             │
│  ③ 空闲                                                     │
│  等待新消息 → 会话数据持久化 → 转录文件保存                    │
│                                                             │
│  ④ 压缩                                                     │
│  上下文过长 → 摘要生成 → 历史替换 → 转录更新                  │
│                                                             │
│  ⑤ 清理                                                     │
│  过期检测 → 转录归档/删除 → 资源释放                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 关键源码文件索引

| 文件路径 | 说明 |
|---------|------|
| `src/routing/session-key.ts` | 会话键构建核心 |
| `src/routing/resolve-route.ts` | 路由到会话键的映射 |
| `src/config/sessions.ts` | 会话存储操作 |
| `src/sessions/input-provenance.ts` | 输入溯源 |
| `src/sessions/level-overrides.ts` | 级别覆盖 |
| `src/sessions/model-overrides.ts` | 模型覆盖 |
| `src/sessions/send-policy.ts` | 发送策略 |
| `src/sessions/session-key-utils.ts` | 会话键工具 |
| `src/sessions/session-label.ts` | 会话标签 |
| `src/sessions/transcript-events.ts` | 转录事件 |
| `src/agents/session-write-lock.ts` | 会话写入锁 |
| `src/agents/session-slug.ts` | 会话 Slug |
| `src/agents/session-file-repair.ts` | 会话文件修复 |
| `src/agents/session-tool-result-guard.ts` | 工具结果保护 |
| `src/gateway/session-utils.ts` | Gateway 会话工具 |
| `src/gateway/server-session-key.ts` | Gateway 会话键解析 |
| `src/gateway/sessions-patch.ts` | 会话补丁 API |
