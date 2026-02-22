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

`getReplyFromConfig()` 是回复系统的核心入口：

```typescript
async function getReplyFromConfig(
  ctx: MsgContext,
  opts?: GetReplyOptions,
  configOverride?: OpenClawConfig,
): Promise<ReplyPayload | ReplyPayload[] | undefined>
```

**回复解析流程**：

1. **会话解析**: 使用 `CommandTargetSessionKey`（原生命令）或回退到 `SessionKey`
2. **模型选择**: 通过 `resolveDefaultModel()` 解析默认供应商/模型，然后依次应用：
   - 心跳模型覆盖 (`opts.isHeartbeat`)
   - 通道模型覆盖 (`resolveChannelModelOverride()`)
   - 重置模型覆盖 (`applyResetModelOverride()`)
3. **工作区设置**: 通过 `ensureAgentWorkspace()` 确保 Agent 工作区就绪
4. **Skill 过滤合并**: `mergeSkillFilters()` 合并通道和 Agent 的 Skill 过滤器（取交集）
5. **打字控制器**: 通过 `createTypingController()` 初始化，默认间隔 6 秒

**回复负载链**：
1. `resolveReplyDirectives()` — 检查是否有需要提前返回的指令回复
2. `handleInlineActions()` — 处理内联命令/技能
3. `stageSandboxMedia()` — 准备沙箱媒体
4. `runPreparedReply()` — 执行主 Agent 回复

**关键回复选项** (`GetReplyOptions`)：

```typescript
type GetReplyOptions = {
  runId?: string;
  abortSignal?: AbortSignal;
  images?: ImageContent[];
  isHeartbeat?: boolean;
  onPartialReply?: (payload: ReplyPayload) => Promise<void> | void;
  onBlockReply?: (payload: ReplyPayload, context?: BlockReplyContext) => Promise<void> | void;
  onToolStart?: (payload: { name?: string; phase?: string }) => Promise<void> | void;
  onModelSelected?: (ctx: ModelSelectedContext) => void;
  skillFilter?: string[];
  timeoutOverrideSeconds?: number;
  // ... 更多回调
};
```

**特殊令牌**：

```typescript
const HEARTBEAT_TOKEN = "HEARTBEAT_OK";   // 心跳回复标记
const SILENT_REPLY_TOKEN = "NO_REPLY";    // 静默回复标记
```

### 3.2 指令解析 (`src/auto-reply/reply/directives.ts`)

用户可在消息中嵌入 `/directive` 格式的指令来控制 Agent 行为：

| 指令 | 别名 | 有效级别 | 说明 |
|------|------|---------|------|
| `/think` | `/t` | `off`, `minimal`, `low`, `medium`, `high`, `xhigh` | 控制推理深度 |
| `/verbose` | `/v` | `off`, `on`, `full` | 详细输出控制 |
| `/notice` | `/notices` | `off`, `on`, `full` | 通知级别 |
| `/elevated` | `/elev` | `off`, `on`, `ask`, `full` | 提升权限 |
| `/reasoning` | `/reason` | `off`, `on`, `stream` | 推理过程输出 |
| `/status` | — | （简单开关） | 状态显示 |

**指令提取模式**：

```typescript
// 匹配模式: (?:^|\s)/directive(?=$|\s|:)(?:\s*:\s*)?level?
// 示例: "/think: high" → { level: "high", hasDirective: true }
// 清理逻辑: 将指令替换为空格，规范化多余空格
```

**综合指令结果** (`InlineDirectives`)：

```typescript
type InlineDirectives = {
  cleaned: string;                // 清理后的消息文本
  hasThinkDirective: boolean;
  thinkLevel?: ThinkLevel;
  hasVerboseDirective: boolean;
  hasReasoningDirective: boolean;
  hasElevatedDirective: boolean;
  hasModelDirective: boolean;     // 模型切换指令
  hasQueueDirective: boolean;     // 队列控制指令
  queueMode?: QueueMode;
  debounceMs?: number;
  // ... 更多字段
};
```

### 3.3 回复标签 (`src/auto-reply/reply/reply-tags.ts`)

`extractReplyToTag()` 从消息中解析 `@reply` 标签，确定回复目标。

**回复指令解析结果**：

```typescript
type ReplyDirectiveParseResult = {
  text: string;
  mediaUrls?: string[];
  replyToId?: string;
  replyToCurrent: boolean;     // [[reply_to_current]] 占位符
  replyToTag: boolean;
  audioAsVoice?: boolean;      // 作为语音气泡发送
  isSilent: boolean;
};
```

### 3.4 模板引擎 (`src/auto-reply/templating.ts`)

`applyTemplate()` 使用 `{{Placeholder}}` 语法支持丰富的变量模板：

```typescript
function applyTemplate(str: string | undefined, ctx: TemplateContext) {
  return str.replace(/{{\s*(\w+)\s*}}/g, (_, key) => {
    const value = ctx[key as keyof TemplateContext];
    return formatTemplateValue(value);
  });
}
```

**可用模板变量**：

| 分类 | 变量 | 说明 |
|------|------|------|
| **发送者** | `From`, `SenderName`, `SenderUsername`, `SenderId`, `SenderTag` | 发送者身份信息 |
| **消息内容** | `Body`, `BodyForAgent`, `CommandBody`, `CommandArgs` | 消息文本 |
| **回复上下文** | `ReplyToSender`, `ReplyToBody`, `ReplyToId` | 被回复消息 |
| **线程** | `ThreadStarterBody`, `ThreadHistoryBody`, `IsFirstThreadTurn` | 线程上下文 |
| **群组** | `GroupSubject`, `GroupChannel`, `GroupSpace`, `GroupMembers` | 群组信息 |
| **媒体** | `MediaPath`, `MediaUrl`, `MediaType`, `MediaPaths[]` | 附件媒体 |
| **会话** | `SessionKey`, `SessionId`, `IsNewSession` | 会话状态 |
| **系统** | `Provider`, `Surface`, `ChatType`, `OriginatingChannel` | 通道信息 |
| **转发** | `ForwardedFromType`, `ForwardedFromId`, `ForwardedFromUsername` | 转发消息源 |

**值格式化规则**：
- `null/undefined` → 空字符串
- `string` → 原样输出
- `Array` → 逗号拼接（跳过 null 和非原始值）
- `object` → 空字符串

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

### 4.3 流式订阅与软分块 (`src/agents/pi-embedded-subscribe.ts`)

`subscribeEmbeddedPiSession()` 管理 LLM 响应流的处理：

- **消息处理** (`handlers.messages.ts`): 文本块拼接、段落分割
- **工具处理** (`handlers.tools.ts`): 工具调用识别与执行
- **压缩处理** (`handlers.compaction.ts`): 自动压缩触发
- **生命周期** (`handlers.lifecycle.ts`): 运行开始/结束事件

**文本流事件类型**：
- `text_start`, `text_delta`, `text_end` — 文本流事件
- `thinking_start`, `thinking_delta`, `thinking_end` — 扩展思考事件
- Delta 去重: 与之前内容比较避免重复发送

**软分块算法** (`EmbeddedBlockChunker`)：

```typescript
type BlockReplyChunking = {
  minChars: number;      // 最小字符数 (默认 800)
  maxChars: number;      // 硬性上限 (默认 1200)
  breakPreference?: "paragraph" | "newline" | "sentence";
  flushOnParagraph?: boolean;  // 段落即刻刷出
};
```

**分块优先级**（从高到低）：
1. **段落** (`\n\n`) — 最安全，保持结构
2. **换行** (`\n`) — 中等优先级
3. **句子** (`[.!?](?=\s|$)`) — 回退选项
4. **硬切割** — 在 maxChars 处强制分割，处理围栏

**围栏代码块安全**：
- 通过 `parseFenceSpans()` 解析代码围栏
- 从不在围栏内部断开（关闭围栏，在下一块重新打开）
- `isSafeFenceBreak()` 验证分割位置
- 维护 `FenceSpan[]` 跟踪各围栏的语言、起止位置

**段落排空模式** (`flushOnParagraph=true`)：
完整段落即使小于 minChars 也立即发出，减少延迟。

## 5. 回复派发

### 5.1 回复派发器 (`src/auto-reply/reply/reply-dispatcher.ts`)

`ReplyDispatcher` 管理回复的序列化发送：

```typescript
type ReplyDispatcher = {
  sendToolResult: (payload: ReplyPayload) => boolean;
  sendBlockReply: (payload: ReplyPayload) => boolean;
  sendFinalReply: (payload: ReplyPayload) => boolean;
  waitForIdle: () => Promise<void>;
  markComplete: () => void;
};
```

**生命周期**（基于 pending 计数器）：

```
初始化: pending = 1 (预留，防止过早触发 idle)
    ↓
sendBlockReply() → pending++ (入队) → deliver() → finally { pending-- }
    ↓
markComplete() → completeCalled = true → pending-- (释放预留)
    ↓
pending === 0 → onIdle() 触发 (Gateway 可重启、资源可清理)
```

**发送链序列化**: 所有回复通过 Promise 链排队，保持 tool → block → final 顺序，块间添加类人延迟。

### 5.2 派发器注册表 (`src/auto-reply/reply/dispatcher-registry.ts`)

全局注册表跟踪所有活跃的派发器：

```typescript
function registerDispatcher(dispatcher: {
  readonly pending: () => number;
  readonly waitForIdle: () => Promise<void>;
}): { id: string; unregister: () => void }

function getTotalPendingReplies(): number
```

确保 Gateway 重启前等待所有活跃回复完成。

### 5.3 出站交付管线 (`src/infra/outbound/deliver.ts`)

```
回复负载 → normalizeReplyPayload()
    → loadChannelOutboundAdapter(channel)
    → 大文本分块 (按通道文本限制)
    → sendText() / sendMedia() (调用通道 API)
    → appendAssistantMessageToSessionTranscript() (会话镜像)
    → triggerInternalHook("message:sent") (钩子触发)
```

### 5.4 通道特定派发

以 Feishu 为例 (`extensions/feishu/src/reply-dispatcher.ts`)：

```
Agent 生成回复 → createFeishuReplyDispatcher()
    → 文本格式化 (Markdown → Feishu 格式)
    → sendMessageFeishu() — API 发送
    → 打字状态管理 (typing.ts)
    → 消息编辑 (editMessageFeishu)
    → 流式卡片更新 (streaming-card.ts)
```

### 5.5 入站去重

`shouldSkipDuplicateInbound()` 通过缓存 `{channel}:{accountId}:{chatId}:{messageId}` 在时间窗口内去除重复消息，防止重放和重复处理。

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

## 7. 通道访问控制

### 7.1 允许列表 (`src/channels/allow-from.ts`)

```typescript
function mergeAllowFromSources(params: {
  allowFrom?: Array<string | number>;
  storeAllowFrom?: string[];
}): string[]

function isSenderIdAllowed(
  allow: { entries: string[]; hasWildcard: boolean; hasEntries: boolean },
  senderId: string | undefined,
  allowWhenEmpty: boolean,
): boolean
// 返回 true: 无条目(allowWhenEmpty)、通配符匹配、或 senderId 在列表中
```

### 7.2 命令门控 (`src/channels/command-gating.ts`)

控制用户是否有权执行文本命令：

```typescript
type CommandGatingModeWhenAccessGroupsOff = "allow" | "deny" | "configured";

function resolveControlCommandGate(params: {
  useAccessGroups: boolean;
  authorizers: CommandAuthorizer[];
  allowTextCommands: boolean;
  hasControlCommand: boolean;
  modeWhenAccessGroupsOff?: CommandGatingModeWhenAccessGroupsOff;
}): { commandAuthorized: boolean; shouldBlock: boolean }
// shouldBlock = allowTextCommands && hasControlCommand && !commandAuthorized
```

**访问组逻辑**：
- `useAccessGroups=false` + `"allow"` → 始终允许
- `useAccessGroups=false` + `"deny"` → 始终拒绝
- `useAccessGroups=false` + `"configured"` → 任一 authorizer 已配置且允许时通过
- `useAccessGroups=true` → 任一 authorizer 已配置且允许时通过

### 7.3 提及门控 (`src/channels/mention-gating.ts`)

控制群组中是否需要 @提及 才响应：

```typescript
function resolveMentionGating(params: {
  requireMention: boolean;
  canDetectMention: boolean;
  wasMentioned: boolean;
  implicitMention?: boolean;
  shouldBypassMention?: boolean;
}): { effectiveWasMentioned: boolean; shouldSkip: boolean }
```

**有效提及** = `wasMentioned || implicitMention || shouldBypassMention`
**跳过** = `requireMention && canDetectMention && !effectiveWasMentioned`

**控制命令绕过**：群组中未被提及但发送了已授权的控制命令时，自动绕过提及要求。

### 7.4 确认反应 (`src/channels/ack-reactions.ts`)

支持通过表情反应确认消息已接收：

```typescript
type AckReactionScope = "all" | "direct" | "group-all" | "group-mentions" | "off" | "none";
```

| 作用域 | 行为 |
|-------|------|
| `all` | 始终反应 |
| `direct` | 仅在私聊中反应 |
| `group-all` | 在所有群组中反应 |
| `group-mentions` | 仅在群组中被提及时反应 |
| `off` / `none` | 从不反应 |

支持回复后自动移除确认反应 (`removeAckReactionAfterReply`)。

## 8. 钩子系统 (`src/hooks/`)

### 8.1 钩子类型

钩子在消息处理管线中提供拦截点：

| 事件类型 | 动作示例 | 说明 |
|---------|---------|------|
| `command` | `new`, `reset` | 命令执行 |
| `session` | `reset`, `bootstrap` | 会话生命周期 |
| `agent` | — | Agent 操作 |
| `gateway` | `startup`, `stop` | Gateway 生命周期 |
| `message` | `received`, `sent` | 消息收发 |

### 8.2 钩子注册与触发

```typescript
function registerInternalHook(eventKey: string, handler: InternalHookHandler): void
// eventKey: "type" 或 "type:action" (如 "command:new", "session:reset")

async function triggerInternalHook(event: InternalHookEvent): Promise<void>
// 调用所有匹配的处理器；错误被捕获并记录，不阻止后续处理器
```

**钩子事件结构**：

```typescript
interface InternalHookEvent {
  type: InternalHookEventType;
  action: string;
  sessionKey: string;
  context: Record<string, unknown>;
  timestamp: Date;
  messages: string[];    // 钩子可向此数组推送消息
}
```

### 8.3 回复前缀 (`src/channels/reply-prefix.ts`)

在多 Agent 环境中，回复可添加 Agent 标识前缀：

```typescript
type ResponsePrefixContext = {
  identityName?: string;    // Agent 名称
  provider?: string;        // LLM 供应商
  model?: string;           // 模型短名称
  modelFull?: string;       // 完整模型标识
  thinkingLevel?: string;   // 思考级别
};
```

前缀在模型选择后通过 `onModelSelected` 回调动态更新。

## 9. 并发控制

### 9.1 Lane 并发系统

`src/agents/lanes.ts` 和 `src/agents/pi-embedded-runner/lanes.ts` 实现了 Lane 并发机制：

- 每个会话键对应一个 Lane
- Lane 内按顺序处理消息
- 跨 Lane 可并行处理
- 全局 Lane 限制防止过载

### 9.2 命令队列 (`src/process/command-queue.ts`)

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
