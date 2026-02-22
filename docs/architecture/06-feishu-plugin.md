# OpenClaw 飞书插件

## 概述

飞书 (Feishu/Lark) 插件是 OpenClaw 的通道扩展之一，实现了与飞书企业消息平台的完整集成。该插件通过 OpenClaw 的插件 SDK 注册为通道插件，支持私聊、群组消息、事件订阅、文档操作、多媒体处理等功能。

## 1. 插件架构

### 1.1 插件注册入口 (`extensions/feishu/index.ts`)

```typescript
const plugin = {
  id: "feishu",
  name: "Feishu",
  description: "Feishu/Lark channel plugin",
  configSchema: emptyPluginConfigSchema(),
  register(api: OpenClawPluginApi) {
    setFeishuRuntime(api.runtime);          // 注入运行时
    api.registerChannel({ plugin: feishuPlugin }); // 注册通道
    registerFeishuDocTools(api);            // 文档工具
    registerFeishuWikiTools(api);           // Wiki 工具
    registerFeishuDriveTools(api);          // 云盘工具
    registerFeishuPermTools(api);           // 权限工具
    registerFeishuBitableTools(api);        // 多维表格工具
  },
};
```

### 1.2 目录结构

```
extensions/feishu/
├── index.ts                 # 插件入口
├── openclaw.plugin.json     # 插件元数据
├── package.json             # 依赖配置
├── skills/                  # 飞书专属技能
└── src/
    ├── accounts.ts          # 多账户管理
    ├── bot.ts               # 机器人消息处理
    ├── channel.ts           # 通道插件定义
    ├── client.ts            # 飞书 API 客户端
    ├── config-schema.ts     # 配置验证
    ├── dedup.ts             # 消息去重
    ├── directory.ts         # 通讯录
    ├── doc-schema.ts        # 文档 API 类型
    ├── docx.ts              # 文档操作
    ├── drive.ts             # 云盘操作
    ├── drive-schema.ts      # 云盘 API 类型
    ├── dynamic-agent.ts     # 动态 Agent 创建
    ├── external-keys.ts     # 外部键映射
    ├── media.ts             # 多媒体处理
    ├── mention.ts           # @提及处理
    ├── monitor.ts           # 消息监听
    ├── onboarding.ts        # 首次配置引导
    ├── outbound.ts          # 出站消息
    ├── perm.ts              # 权限管理
    ├── perm-schema.ts       # 权限 API 类型
    ├── policy.ts            # 消息策略
    ├── probe.ts             # 连接探测
    ├── reactions.ts         # 表情回应
    ├── reply-dispatcher.ts  # 回复派发
    ├── runtime.ts           # 运行时管理
    ├── send.ts              # 消息发送
    ├── send-result.ts       # 发送结果
    ├── streaming-card.ts    # 流式卡片
    ├── targets.ts           # 目标解析
    ├── tools-config.ts      # 工具配置
    ├── types.ts             # 类型定义
    ├── typing.ts            # 打字状态
    ├── wiki.ts              # Wiki 操作
    └── wiki-schema.ts       # Wiki API 类型
```

## 2. 通道定义 (`src/channel.ts`)

### 2.1 通道元数据

```typescript
const meta: ChannelMeta = {
  id: "feishu",
  label: "Feishu",
  selectionLabel: "Feishu/Lark (飞书)",
  docsPath: "/channels/feishu",
  aliases: ["lark"],           // 支持 "lark" 作为别名
  order: 70,
};
```

### 2.2 通道能力

```typescript
capabilities: {
  chatTypes: ["direct", "channel"],  // 支持私聊和群组
  polls: false,                      // 不支持投票
  threads: true,                     // 支持消息线程
  media: true,                       // 支持多媒体
  reactions: true,                   // 支持表情回应
  edit: true,                        // 支持消息编辑
  reply: true,                       // 支持消息回复
}
```

### 2.3 配对 (Pairing) 机制

飞书用户需要通过配对授权才能与 Agent 交互：

```typescript
pairing: {
  idLabel: "feishuUserId",
  normalizeAllowEntry: (entry) => entry.replace(/^(feishu|user|open_id):/i, ""),
  notifyApproval: async ({ cfg, id }) => {
    await sendMessageFeishu({ cfg, to: id, text: PAIRING_APPROVED_MESSAGE });
  },
}
```

## 3. 消息监听 (`src/monitor.ts`)

### 3.1 监听模式

飞书插件支持两种消息接收模式：

**WebSocket 模式**：
```typescript
const wsClients = new Map<string, Lark.WSClient>();

// 使用 Lark SDK 的 WSClient 建立长连接
const ws = createFeishuWSClient(account);
```

**Webhook 模式**：
```typescript
const httpServers = new Map<string, http.Server>();

// 启动 HTTP 服务器接收飞书推送
// 支持 URL 验证 (challenge)
// 支持事件签名验证
```

### 3.2 Webhook 安全机制

```typescript
const FEISHU_WEBHOOK_MAX_BODY_BYTES = 1024 * 1024;     // 1MB 请求体限制
const FEISHU_WEBHOOK_BODY_TIMEOUT_MS = 30_000;          // 30s 超时
const FEISHU_WEBHOOK_RATE_LIMIT_WINDOW_MS = 60_000;     // 60s 速率窗口
const FEISHU_WEBHOOK_RATE_LIMIT_MAX_REQUESTS = 120;     // 120 请求/窗口
```

- 请求体大小限制 (`installRequestBodyLimitGuard`)
- 速率限制 (`isWebhookRateLimited`)
- JSON Content-Type 验证
- 状态码异常计数和日志

### 3.3 事件处理

监听启动后处理以下事件：

| 事件 | 处理函数 | 说明 |
|------|---------|------|
| `im.message.receive_v1` | `handleFeishuMessage` | 接收新消息 |
| Bot Added | `handleBotAddedEvent` | 机器人被添加到群组 |

### 3.4 多账户支持

```typescript
function listEnabledFeishuAccounts(config): ResolvedFeishuAccount[]

// 每个账户独立的:
// - WebSocket 客户端
// - HTTP 服务器
// - Bot OpenID 缓存
```

## 4. 机器人消息处理 (`src/bot.ts`)

### 4.1 消息处理流程

```
飞书推送 → handleFeishuMessage()
    → 消息去重 (dedup.ts)
    → 提取消息体 (mention.ts)
    → 检查 @提及 (checkBotMentioned)
    → 允许列表检查 (policy.ts)
    → 发送者名称解析 (缓存 10 分钟)
    → 群组配置检查 (resolveFeishuGroupConfig)
    → 消息下载附件 (media.ts)
    → 动态 Agent 创建 (dynamic-agent.ts)
    → 构建历史上下文
    → 排入 Agent 运行器
```

### 4.2 发送者名称缓存

```typescript
const SENDER_NAME_TTL_MS = 10 * 60 * 1000;  // 10 分钟缓存
const senderNameCache = new Map<string, { name: string; expireAt: number }>();
```

### 4.3 权限错误处理

飞书 API 返回权限错误 (code: 99991672) 时：

```typescript
function extractPermissionError(err: unknown): PermissionError | null {
  // 提取权限授权 URL
  // 5 分钟冷却期防止重复通知
}
const PERMISSION_ERROR_COOLDOWN_MS = 5 * 60 * 1000;
```

### 4.4 @提及处理 (`src/mention.ts`)

```typescript
function extractMentionTargets(message): MentionTarget[]
function extractMessageBody(message): string
function isMentionForwardRequest(message): boolean
function buildMentionedMessage(text, mentions): string
```

支持：
- 用户 @提及
- 全员 @提及
- 机器人 @提及识别
- 提及转发请求

## 5. 消息发送 (`src/send.ts`)

### 5.1 核心发送函数

```typescript
// 发送文本消息
async function sendMessageFeishu(params: {
  cfg: ClawdbotConfig;
  to: string;
  text: string;
  replyTo?: string;
}): Promise<SendResult>

// 发送卡片消息
async function sendCardFeishu(params: {
  cfg: ClawdbotConfig;
  to: string;
  card: FeishuCard;
}): Promise<SendResult>

// 更新卡片
async function updateCardFeishu(params): Promise<void>

// 编辑消息
async function editMessageFeishu(params): Promise<void>

// 获取消息
async function getMessageFeishu(params): Promise<FeishuMessage>
```

### 5.2 目标解析 (`src/targets.ts`)

```typescript
function normalizeFeishuTarget(target: string): NormalizedTarget
function looksLikeFeishuId(value: string): boolean
function formatFeishuTarget(target: NormalizedTarget): string
```

目标格式：
- `user:ou_xxxxx` — 用户 (open_id)
- `chat:oc_xxxxx` — 群组 (chat_id)
- 直接使用 open_id/chat_id

## 6. 回复派发 (`src/reply-dispatcher.ts`)

### 6.1 流式回复

`createFeishuReplyDispatcher()` 创建回复派发器，支持：

- **文本流式发送**: 将 Agent 的流式输出实时发送
- **卡片流式更新**: 使用飞书交互卡片实现打字效果
- **消息聚合**: 短消息聚合后批量发送

### 6.2 流式卡片 (`src/streaming-card.ts`)

使用飞书的交互卡片实现类似 ChatGPT 的流式输出效果：

1. 发送初始卡片（显示"正在思考..."）
2. 通过 `updateCardFeishu()` 实时更新卡片内容
3. 完成后更新为最终状态

### 6.3 打字状态 (`src/typing.ts`)

在 Agent 处理消息时显示"正在输入"状态。

## 7. 多媒体处理 (`src/media.ts`)

### 7.1 上传

```typescript
async function uploadImageFeishu(params): Promise<string>
async function uploadFileFeishu(params): Promise<string>
async function sendImageFeishu(params): Promise<SendResult>
async function sendFileFeishu(params): Promise<SendResult>
async function sendMediaFeishu(params): Promise<SendResult>
```

### 7.2 下载

```typescript
async function downloadMessageResourceFeishu(params): Promise<Buffer>
```

支持处理消息中的图片、文件、音频、视频附件。

## 8. 表情回应 (`src/reactions.ts`)

```typescript
async function addReactionFeishu(params): Promise<void>
async function removeReactionFeishu(params): Promise<void>
async function listReactionsFeishu(params): Promise<Reaction[]>

enum FeishuEmoji {
  // 飞书内置表情映射
}
```

用于：
- 消息确认反应 (Ack)
- 处理状态指示
- 错误状态通知

## 9. 飞书文档/云盘/Wiki 工具

### 9.1 文档操作 (`src/docx.ts`)

通过 Agent 工具操作飞书文档：

- 创建文档
- 读取文档内容
- 更新文档
- 文档格式转换

### 9.2 云盘操作 (`src/drive.ts`)

- 列出文件
- 上传/下载文件
- 文件夹操作
- 权限管理

### 9.3 Wiki 操作 (`src/wiki.ts`)

- 知识库页面操作
- 搜索 Wiki 内容
- 创建/更新 Wiki 页面

### 9.4 多维表格 (`src/bitable.ts`)

- 读取/写入多维表格记录
- 查询和过滤
- 字段操作

### 9.5 权限管理 (`src/perm.ts`)

- 文档/云盘权限查询
- 权限授予/撤回

## 10. 飞书 API 客户端 (`src/client.ts`)

### 10.1 客户端创建

```typescript
function createFeishuClient(account: ResolvedFeishuAccount): LarkClient
function createFeishuWSClient(account: ResolvedFeishuAccount): Lark.WSClient
function createEventDispatcher(account: ResolvedFeishuAccount): EventDispatcher
```

使用 `@larksuiteoapi/node-sdk` 官方 SDK。

### 10.2 账户解析 (`src/accounts.ts`)

```typescript
function resolveFeishuAccount(config, accountId?): ResolvedFeishuAccount
function resolveFeishuCredentials(config, accountId?): FeishuCredentials
function listFeishuAccountIds(config): string[]
function resolveDefaultFeishuAccountId(config): string
```

支持多飞书应用配置：

```yaml
channels:
  feishu:
    enabled: true
    appId: "cli_xxx"
    appSecret: "xxx"
    accounts:
      - id: "account1"
        appId: "cli_yyy"
        appSecret: "yyy"
```

## 11. 消息策略 (`src/policy.ts`)

### 11.1 群组消息策略

```typescript
function resolveFeishuGroupConfig(config, chatId): GroupConfig
function resolveFeishuReplyPolicy(config, chatId): ReplyPolicy
function isFeishuGroupAllowed(config, chatId): boolean
function resolveFeishuAllowlistMatch(config, senderId): boolean
```

控制：
- 哪些群组允许 Agent 响应
- 是否需要 @提及才回复
- 群组级别的工具权限

### 11.2 工具策略

```typescript
function resolveFeishuGroupToolPolicy(config, chatId): ToolPolicy
```

群组可以限制 Agent 可使用的工具。

## 12. 连接探测 (`src/probe.ts`)

```typescript
async function probeFeishu(account): Promise<ProbeResult>
```

验证飞书应用凭证和连接状态：
- 获取 Bot OpenID
- 验证 API 访问权限
- 检查 Token 有效性

## 13. 动态 Agent 创建 (`src/dynamic-agent.ts`)

支持根据群组或消息上下文动态创建专属 Agent 实例：

```typescript
function maybeCreateDynamicAgent(params: DynamicAgentCreationConfig): Agent | null
```

## 14. 数据流图

```
┌──────────────────────────────────────────────────────────┐
│                    飞书消息流                              │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────┐    ┌───────────────┐                    │
│  │ 飞书服务器  │───→│ WebSocket/    │                    │
│  │            │    │ Webhook       │                    │
│  └────────────┘    └───────┬───────┘                    │
│                            │                            │
│                    ┌───────▼───────┐                    │
│                    │ monitor.ts    │                    │
│                    │ 事件分发       │                    │
│                    └───────┬───────┘                    │
│                            │                            │
│            ┌───────────────┼───────────────┐            │
│            ▼               ▼               ▼            │
│    ┌──────────────┐ ┌──────────┐ ┌──────────────┐      │
│    │ 消息去重     │ │ @提及    │ │ 允许列表     │      │
│    │ (dedup)      │ │ 检查     │ │ 检查         │      │
│    └──────┬───────┘ └────┬─────┘ └──────┬───────┘      │
│           └──────────────┼──────────────┘               │
│                          ▼                              │
│                  ┌───────────────┐                      │
│                  │ handleFeishu  │                      │
│                  │ Message()     │                      │
│                  └───────┬───────┘                      │
│                          │                              │
│              ┌───────────▼───────────┐                  │
│              │ OpenClaw Agent 运行器  │                  │
│              └───────────┬───────────┘                  │
│                          │                              │
│              ┌───────────▼───────────┐                  │
│              │ reply-dispatcher.ts   │                  │
│              └───────────┬───────────┘                  │
│                          │                              │
│            ┌─────────────┼─────────────┐                │
│            ▼             ▼             ▼                │
│    ┌──────────────┐ ┌──────────┐ ┌──────────────┐      │
│    │ send.ts      │ │ media.ts │ │ reactions.ts │      │
│    │ 文本/卡片    │ │ 图片/文件│ │ 表情回应     │      │
│    └──────┬───────┘ └────┬─────┘ └──────┬───────┘      │
│           └──────────────┼──────────────┘               │
│                          ▼                              │
│                  ┌───────────────┐                      │
│                  │ 飞书 API      │                      │
│                  │ (@larksuit..  │                      │
│                  │ node-sdk)     │                      │
│                  └───────────────┘                      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

## 关键源码文件索引

| 文件路径 | 说明 |
|---------|------|
| `extensions/feishu/index.ts` | 插件入口 |
| `extensions/feishu/src/channel.ts` | 通道定义与能力声明 |
| `extensions/feishu/src/monitor.ts` | 消息监听 (WS/Webhook) |
| `extensions/feishu/src/bot.ts` | 消息处理核心 |
| `extensions/feishu/src/client.ts` | 飞书 API 客户端 |
| `extensions/feishu/src/accounts.ts` | 多账户管理 |
| `extensions/feishu/src/send.ts` | 消息发送 |
| `extensions/feishu/src/media.ts` | 多媒体处理 |
| `extensions/feishu/src/mention.ts` | @提及解析 |
| `extensions/feishu/src/reply-dispatcher.ts` | 回复派发 |
| `extensions/feishu/src/streaming-card.ts` | 流式卡片 |
| `extensions/feishu/src/policy.ts` | 消息策略 |
| `extensions/feishu/src/reactions.ts` | 表情回应 |
| `extensions/feishu/src/docx.ts` | 文档操作 |
| `extensions/feishu/src/drive.ts` | 云盘操作 |
| `extensions/feishu/src/wiki.ts` | Wiki 操作 |
| `extensions/feishu/src/bitable.ts` | 多维表格 |
| `extensions/feishu/src/perm.ts` | 权限管理 |
| `extensions/feishu/src/dedup.ts` | 消息去重 |
| `extensions/feishu/src/dynamic-agent.ts` | 动态 Agent |
| `extensions/feishu/src/probe.ts` | 连接探测 |
| `extensions/feishu/src/types.ts` | 类型定义 |
