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

### 2.1 会话存储接口 (`src/config/sessions.ts`)

```typescript
function loadSessionStore(): SessionStore
function saveSessionStore(store: SessionStore): void
function resolveStorePath(): string
function resolveSessionKey(params): string
function deriveSessionKey(params): string
```

### 2.2 存储路径

会话数据存储在文件系统中：

```
~/.config/openclaw/
├── config.yaml              # 全局配置
├── sessions/
│   ├── agent-default/
│   │   ├── main/
│   │   │   ├── transcript.json   # 会话转录
│   │   │   └── metadata.json     # 会话元数据
│   │   ├── discord:default:direct:123/
│   │   └── feishu:acct1:channel:456/
│   └── agent-vip/
└── auth/
    └── profiles.json        # 认证配置
```

### 2.3 会话数据结构

每个会话包含：

- **transcript.json**: 完整的消息转录（AgentMessage 数组）
- **metadata.json**: 会话元数据（创建时间、最后活跃时间等）
- **工具状态**: 工具执行的持久化状态

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

防止对同一会话的并发写入：

```typescript
// 会话写入锁确保同一时刻只有一个运行可以修改会话文件
// 使用文件锁或内存锁实现
```

### 3.4 会话暂停与恢复

- 会话在 Gateway 重启后自动恢复
- 子 Agent 会话可以被父 Agent 暂停/恢复
- 未完成的工具调用在恢复后重新执行

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
