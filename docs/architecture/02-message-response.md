# OpenClaw 消息响应回复机制

## 概述

OpenClaw 的消息响应系统实现了从多通道消息接收到 AI Agent 处理再到回复派发的完整流水线。该系统采用事件驱动架构，通过路由绑定将消息定向到正确的 Agent 会话。

## 1. 消息接收入口

### 1.1 通道监听

每个消息通道 (Channel) 通过 `monitor*` 函数启动消息监听：

- **Feishu**: `monitorFeishuProvider()` — WebSocket/Webhook 监听
- **Discord**: `monitorDiscordProvider()` — Discord.js 客户端
- **Telegram**: `monitorTelegramProvider()` — 长轮询/Webhook
- **Slack**: `monitorSlackProvider()` — Socket Mode/Events API
- **Web**: `monitorWebChannel()` — HTTP/WebSocket 端点

监听由 Gateway 服务器在启动时统一初始化 (`src/gateway/server-channels.ts`)。

### 1.2 消息事件结构

通道接收到的原始消息被标准化为统一格式：

```typescript
type IncomingMessage = {
  channel: string;        // 通道标识 (feishu, discord, telegram...)
  accountId: string;      // 账户 ID
  peer: RoutePeer;        // 对话方 { kind: ChatType, id: string }
  text: string;           // 消息文本
  messageId?: string;     // 原始消息 ID
  threadId?: string;      // 线程 ID (群组消息)
  guildId?: string;       // 服务器/群组 ID
  teamId?: string;        // 团队 ID
  attachments?: [];       // 附件
  senderName?: string;    // 发送者名称
};
```

### 1.3 ChatType 类型 (`src/channels/chat-type.ts`)

```typescript
type ChatType = "direct" | "channel" | "thread";
```

- **direct**: 一对一私聊
- **channel**: 群组/频道消息
- **thread**: 线程消息 (群组中的回复线程)

## 2. 消息路由

### 2.1 路由解析 (`src/routing/resolve-route.ts`)

`resolveAgentRoute()` 是路由核心函数，接收消息元数据，返回路由结果：

```typescript
type ResolveAgentRouteInput = {
  cfg: OpenClawConfig;
  channel: string;
  accountId?: string | null;
  peer?: RoutePeer | null;
  parentPeer?: RoutePeer | null;  // 线程父对话方
  guildId?: string | null;
  teamId?: string | null;
  memberRoleIds?: string[];       // Discord 角色
};

type ResolvedAgentRoute = {
  agentId: string;                // 目标 Agent ID
  channel: string;                // 通道
  accountId: string;              // 账户 ID
  sessionKey: string;             // 会话键
  mainSessionKey: string;         // 主会话键
  matchedBy: string;              // 匹配方式描述
};
```

### 2.2 绑定匹配优先级

路由通过配置中的 **绑定 (Bindings)** 系统实现，匹配按以下优先级从高到低执行：

1. **binding.peer** — 精确对话方匹配 (用户 ID / 频道 ID)
2. **binding.peer.parent** — 线程父对话方匹配继承
3. **binding.guild+roles** — 服务器 + 角色组合匹配
4. **binding.guild** — 服务器匹配 (无角色约束)
5. **binding.team** — 团队匹配
6. **binding.account** — 账户匹配 (非通配符)
7. **binding.channel** — 通道匹配 (通配符账户)
8. **default** — 默认 Agent

```typescript
// 绑定配置示例
bindings: [
  {
    match: { channel: "discord", guildId: "123", roles: ["admin"] },
    agentId: "admin-agent"
  },
  {
    match: { channel: "feishu", peer: { kind: "direct", id: "user123" } },
    agentId: "vip-agent"
  }
]
```

### 2.3 会话键构建

路由解析后生成唯一的 `sessionKey`，格式为：

```
{agentId}:{channel}:{accountId}:{peerKind}:{peerId}
```

支持多种 DM 作用域模式 (`dmScope`)：
- **main**: 所有 DM 共享一个会话
- **per-peer**: 每个对话方独立会话
- **per-channel-peer**: 每个通道+对话方独立会话
- **per-account-channel-peer**: 每个账户+通道+对话方独立会话

## 3. 自动回复系统

### 3.1 回复获取 (`src/auto-reply/reply/get-reply.ts`)

`getReplyFromConfig()` 从配置中解析回复参数：

- 提取消息中的指令标记 (directives)
- 解析回复目标
- 应用模板引擎

### 3.2 指令解析 (`src/auto-reply/reply/directives.ts`)

支持从消息中提取特殊指令：

- `extractThinkDirective()`: 提取 `!think` 指令控制推理深度
- `extractReasoningDirective()`: 推理级别指令
- `extractElevatedDirective()`: 提升权限指令
- `extractVerboseDirective()`: 详细输出指令
- `extractExecDirective()`: 执行命令指令
- `extractQueueDirective()`: 队列指令

### 3.3 回复标签 (`src/auto-reply/reply/reply-tags.ts`)

`extractReplyToTag()` 从消息中解析 `@reply` 标签，确定回复目标。

### 3.4 模板引擎 (`src/auto-reply/templating.ts`)

`applyTemplate()` 支持在回复中使用变量模板：

- `{{sender}}` — 发送者名称
- `{{channel}}` — 当前通道
- `{{time}}` — 当前时间
- 自定义变量

## 4. Agent 消息处理

### 4.1 消息排队

消息通过 `queueEmbeddedPiMessage()` 进入 Agent 运行器：

```
消息到达 → 路由解析 → 会话键确定
    → queueEmbeddedPiMessage(sessionKey, message)
        → 检查是否有活跃运行
            → 有: 消息入队等待
            → 无: 启动新的 runEmbeddedPiAgent()
```

### 4.2 嵌入式运行 (`runEmbeddedPiAgent`)

完整的 Agent 运行流程：

1. **模型解析**: `resolveModel()` — 确定使用的 LLM 模型和供应商
2. **认证获取**: `getApiKeyForModel()` — 获取 API 密钥，支持认证配置轮换
3. **负载构建**: `buildEmbeddedRunPayloads()` — 组装 LLM 请求
4. **系统提示词**: 注入当前上下文、技能、工具说明
5. **执行尝试**: `runEmbeddedAttempt()` — 调用 LLM API
6. **流式处理**: 实时处理 LLM 响应流
7. **工具调用**: 执行 LLM 请求的工具操作
8. **结果处理**: 收集完整响应并持久化

### 4.3 流式订阅 (`src/agents/pi-embedded-subscribe.ts`)

`subscribeEmbeddedPiSession()` 管理 LLM 响应流的处理：

- **消息处理** (`handlers.messages.ts`): 文本块拼接、段落分割
- **工具处理** (`handlers.tools.ts`): 工具调用识别与执行
- **压缩处理** (`handlers.compaction.ts`): 自动压缩触发
- **生命周期** (`handlers.lifecycle.ts`): 运行开始/结束事件

流式输出优化：
- 软分块 (soft chunks): 按段落偏好分割长回复
- 代码块感知: 保持围栏代码块完整
- 重新打开代码块: 跨分块边界时自动重新打开

## 5. 回复派发

### 5.1 回复派发器注册表 (`src/auto-reply/reply/dispatcher-registry.ts`)

- `getTotalPendingReplies()`: 获取全局待发送回复数
- 每个通道有独立的回复派发器

### 5.2 通道特定派发

以 Feishu 为例 (`extensions/feishu/src/reply-dispatcher.ts`)：

```
Agent 生成回复 → createFeishuReplyDispatcher()
    → 文本格式化 (Markdown → Feishu 格式)
    → sendMessageFeishu() — API 发送
    → 打字状态管理 (typing.ts)
    → 消息编辑 (editMessageFeishu)
    → 流式卡片更新 (streaming-card.ts)
```

### 5.3 回复前缀 (`src/channels/reply-prefix.ts`)

在多 Agent 环境中，回复可添加 Agent 标识前缀。

### 5.4 Ack 反应 (`src/channels/ack-reactions.ts`)

部分通道支持"已收到"反应（如表情回应），表示消息已被处理。

## 6. 消息生命周期图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    消息生命周期完整流程                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ① 消息接收                                                         │
│  ┌───────────┐    ┌──────────────┐    ┌────────────────┐           │
│  │ 通道监听   │───→│ 消息标准化    │───→│ 允许列表检查    │           │
│  │ (monitor)  │    │ (normalize)  │    │ (allowlist)    │           │
│  └───────────┘    └──────────────┘    └───────┬────────┘           │
│                                               │                     │
│  ② 路由                                      ▼                     │
│  ┌───────────┐    ┌──────────────┐    ┌────────────────┐           │
│  │ 绑定匹配   │←──│ 路由解析      │←──│ 命令/提及门控   │           │
│  │ (bindings) │    │ (route)      │    │ (gating)       │           │
│  └─────┬─────┘    └──────────────┘    └────────────────┘           │
│        │                                                            │
│  ③ Agent 处理                                                       │
│        ▼                                                            │
│  ┌───────────┐    ┌──────────────┐    ┌────────────────┐           │
│  │ 消息排队   │───→│ 构建 LLM 请求 │───→│ 执行 LLM 调用   │          │
│  │ (queue)    │    │ (payloads)   │    │ (attempt)      │           │
│  └───────────┘    └──────────────┘    └───────┬────────┘           │
│                                               │                     │
│  ④ 工具循环                                   ▼                     │
│  ┌───────────┐    ┌──────────────┐    ┌────────────────┐           │
│  │ 工具执行   │←──│ 工具调用识别   │←──│ 流式响应处理    │           │
│  │ (execute)  │    │ (parse)      │    │ (subscribe)    │           │
│  └─────┬─────┘    └──────────────┘    └────────────────┘           │
│        │  ↕ (循环直到 LLM 给出最终回复)                              │
│                                                                     │
│  ⑤ 回复派发                                                         │
│  ┌───────────┐    ┌──────────────┐    ┌────────────────┐           │
│  │ 格式化     │───→│ 分块/流式发送 │───→│ 通道 API 发送   │           │
│  │ (format)   │    │ (dispatch)   │    │ (send)         │           │
│  └───────────┘    └──────────────┘    └────────────────┘           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## 7. 钩子系统 (`src/hooks/`)

### 7.1 全局钩子

钩子在消息处理管线中提供拦截点：

- **before-tool-call**: 工具执行前触发
- **after-tool-call**: 工具执行后触发
- **gateway-stop**: Gateway 停止前触发

### 7.2 钩子运行器 (`src/plugins/hook-runner-global.ts`)

`getGlobalHookRunner()` 返回全局钩子运行器，执行所有已注册的钩子。

## 8. 并发控制

### 8.1 Lane 并发系统

`src/agents/lanes.ts` 和 `src/agents/pi-embedded-runner/lanes.ts` 实现了 Lane 并发机制：

- 每个会话键对应一个 Lane
- Lane 内按顺序处理消息
- 跨 Lane 可并行处理
- 全局 Lane 限制防止过载

### 8.2 命令队列 (`src/process/command-queue.ts`)

通过 `enqueueCommandInLane()` 将 Agent 运行排入队列：

- 基于 Lane 的 FIFO 排队
- 全局队列大小监控
- 支持异步等待队列完成

## 关键源码文件索引

| 文件路径 | 说明 |
|---------|------|
| `src/routing/resolve-route.ts` | 路由解析核心 |
| `src/routing/bindings.ts` | 绑定配置解析 |
| `src/routing/session-key.ts` | 会话键构建 |
| `src/auto-reply/reply.ts` | 回复系统入口 |
| `src/auto-reply/reply/get-reply.ts` | 回复参数获取 |
| `src/auto-reply/reply/directives.ts` | 指令解析 |
| `src/auto-reply/templating.ts` | 模板引擎 |
| `src/agents/pi-embedded-subscribe.ts` | 流式响应订阅 |
| `src/agents/pi-embedded-subscribe.handlers.messages.ts` | 消息流处理 |
| `src/agents/pi-embedded-subscribe.handlers.tools.ts` | 工具流处理 |
| `src/channels/chat-type.ts` | 聊天类型定义 |
| `src/channels/allow-from.ts` | 允许列表 |
| `src/channels/command-gating.ts` | 命令门控 |
| `src/channels/mention-gating.ts` | 提及门控 |
