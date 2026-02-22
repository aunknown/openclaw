# OpenClaw Gateway

## 概述

OpenClaw Gateway 是系统的核心服务组件，作为 HTTP/WebSocket 服务器运行，负责桥接所有消息通道与 AI Agent。Gateway 提供 REST API、WebSocket 实时通信、通道管理、插件加载、定时任务等功能。

## 1. Gateway 架构

### 1.1 核心组件

```
┌─────────────────────────────────────────────────────────────┐
│                     OpenClaw Gateway                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  HTTP Server  │  │  WS Server   │  │ Control UI   │      │
│  │  (REST API)   │  │ (实时通信)    │  │ (管理界面)    │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         └──────────────────┼──────────────────┘              │
│                            │                                 │
│  ┌──────────────┐  ┌──────▼───────┐  ┌──────────────┐      │
│  │  Channel     │  │  Method      │  │  Plugin      │      │
│  │  Manager     │  │  Handlers    │  │  Registry    │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         │                  │                  │              │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐      │
│  │  Agent       │  │  Node        │  │  Cron        │      │
│  │  Event Hndlr │  │  Registry    │  │  Service     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 入口与启动 (`src/gateway/server.impl.ts`)

`startGatewayServer()` 是 Gateway 的主启动函数：

```typescript
type GatewayServerOptions = {
  bind?: "loopback" | "all" | "tailscale";  // 绑定地址
  port?: number;                             // 端口号
  force?: boolean;                           // 强制启动
  runtime?: RuntimeEnv;
  tlsCert?: string;                          // TLS 证书
  tlsKey?: string;                           // TLS 私钥
};

type GatewayServer = {
  close: (opts?: {
    reason?: string;
    restartExpectedMs?: number | null;
  }) => Promise<void>;
};
```

### 1.3 启动流程

```
startGatewayServer()
    → migrateLegacyConfig()       // 迁移旧配置
    → loadConfig()                // 加载配置
    → ensureGatewayStartupAuth()  // 认证初始化
    → loadGatewayPlugins()        // 加载插件
    → loadGatewayModelCatalog()   // 加载模型目录
    → createChannelManager()      // 创建通道管理器
    → createAgentEventHandler()   // 创建 Agent 事件处理器
    → startGatewayDiscovery()     // 启动服务发现
    → attachGatewayWsHandlers()   // 附加 WS 处理器
    → buildGatewayCronService()   // 构建定时任务
    → startGatewaySidecars()      // 启动辅助服务
    → startChannelHealthMonitor() // 启动通道健康监控
    → startGatewayConfigReloader()// 启动配置热重载
    → logGatewayStartup()         // 输出启动日志
```

## 2. HTTP 服务器与 API 端点

### 2.1 HTTP 基础 (`src/gateway/server-http.ts`)

Gateway 在单个端口（默认 `18789`）上多路复用 HTTP 和 WebSocket：

| 端点 | 方法 | 用途 | 文件 |
|------|------|------|------|
| `/v1/chat/completions` | POST | OpenAI 兼容聊天 API | `openai-http.ts` |
| `/v1/responses` | POST | OpenResponses 协议 | `openresponses-http.ts` |
| `/tools/invoke` | POST | 直接工具调用 | `tools-invoke-http.ts` |
| `/hooks` | POST | Webhook 消息注入 | `hooks.ts` |
| `/__openclaw__/canvas/*` | GET/POST | Canvas 浏览器自动化 | `server-browser.ts` |
| `/avatar/*` | GET | Agent 头像 | `control-ui.ts` |
| `/health` | GET | 健康检查 | `probe.ts` |
| `/` | GET | Control UI SPA | `control-ui.ts` |

### 2.2 认证系统 (`src/gateway/auth.ts`)

Gateway 支持 5 种认证模式：

| 模式 | 配置 | 说明 |
|------|------|------|
| **Token** (默认) | `gateway.auth.token` | Bearer Token 认证，时间安全比较 |
| **Password** | `gateway.auth.password` | 密码认证 |
| **Trusted Proxy** | `gateway.auth.trustedProxy` | 可信代理 IP + 用户头部 |
| **Tailscale** | `gateway.tailscale.mode="serve"` | Tailscale 用户验证 |
| **None** | `gateway.auth.mode="none"` | 无认证 (仅限 loopback) |

核心函数：
- `resolveGatewayAuth()` (line 186): 解析认证配置
- `authorizeGatewayConnect()` (line 322): 验证连接请求

**设备认证与配对** (`device-auth.ts`):
- 节点发送设备指纹 + 公钥
- 挑战-响应签名验证
- 按角色+作用域颁发设备 Token
- 新设备需要配对审批

**认证速率限制** (`auth-rate-limit.ts`):
```typescript
function createAuthRateLimiter(): AuthRateLimiter
// 可配置的暴力破解防护
// 超出限制返回 429 + Retry-After 头
```

**方法级权限** (`method-scopes.ts`):
- `operator.read` — 只读访问
- `operator.write` — 修改配置/会话
- `operator.admin` — 完全访问
- `operator.approvals` — 审批执行请求
- `operator.pairing` — 管理设备配对

## 3. WebSocket 协议

### 3.1 握手序列

Gateway WS 协议使用挑战-响应式握手（协议版本 3）：

```
服务器 → 客户端:  connect.challenge { nonce, ts }
客户端 → 服务器:  connect { auth, role, scopes, device, client }
服务器 → 客户端:  hello-ok { protocol, policy, auth: { deviceToken, role } }
```

### 3.2 帧类型

```typescript
// 请求帧
{ type: "req", id: string, method: string, params?: object }

// 响应帧
{ type: "res", id: string, ok: boolean, payload?: object, error?: { code, message } }

// 事件帧 (服务器推送)
{ type: "event", event: string, payload?: object, seq?: number, stateVersion?: object }
```

### 3.3 WS 方法 (100+ RPC 方法)

Gateway 通过 WS 暴露丰富的 RPC 方法：

| 类别 | 方法示例 | 说明 |
|------|---------|------|
| **聊天** | `chat.send`, `chat.history`, `chat.abort` | 消息发送/历史/中止 |
| **Agent** | `agent`, `agent.wait`, `agent.identity` | 运行/等待/身份 |
| **会话** | `sessions.reset`, `sessions.list`, `sessions.describe` | 会话管理 |
| **通道** | `channels.status`, `channels.login`, `channels.logout` | 通道操作 |
| **配置** | `config.get`, `config.apply`, `config.patch` | 配置读写 |
| **健康** | `health`, `status` | 状态查询 |
| **节点** | `node.pair.request`, `node.invoke`, `node.describe` | 移动/远程节点 |
| **审批** | `exec.approval.list`, `exec.approval.resolve` | 工具执行审批 |
| **技能** | `skills.list`, `skills.bins`, `skills.update` | 技能管理 |
| **定时** | `cron.schedule`, `cron.list` | 定时任务 |
| **日志** | `logs.subscribe`, `logs.search` | 日志流/搜索 |

### 3.4 服务器广播 (`src/gateway/server-broadcast.ts`)

向所有连接的 WS 客户端广播实时事件（Agent 进度、通道状态变更等）。

## 4. API 方法系统

### 4.1 方法列表 (`src/gateway/server-methods-list.ts`)

`listGatewayMethods()` 返回所有注册的 API 方法和 `GATEWAY_EVENTS` 事件列表。

### 4.2 核心方法处理 (`src/gateway/server-methods.ts`)

`coreGatewayHandlers()` 注册核心 API 处理器：

- 会话管理
- 配置操作
- 通道状态
- 模型查询
- 节点管理

### 4.3 方法作用域 (`src/gateway/method-scopes.ts`)

控制 API 方法的访问权限：

```typescript
// 每个方法有定义的作用域
// 不同角色有不同的方法访问权限
```

### 4.4 OpenAI 兼容 API (`src/gateway/openai-http.ts`)

提供 OpenAI Chat Completions 兼容的 HTTP 接口：

```
POST /v1/chat/completions
POST /v1/responses
```

### 4.5 Open Responses API (`src/gateway/openresponses-http.ts`)

实现 Open Responses 协议标准。

## 5. 通道管理

### 5.1 通道管理器 (`src/gateway/server-channels.ts`)

`createChannelManager()` 管理所有消息通道：

- 通道注册与初始化
- 通道健康监控
- 通道启停控制
- 通道状态查询

### 5.2 通道健康监控 (`src/gateway/channel-health-monitor.ts`)

`startChannelHealthMonitor()` 定期检查通道连接状态：

```typescript
// 定期探测每个通道的连接状态
// 记录健康状态变更
// 触发告警或自动恢复
```

## 6. Agent 事件处理

### 6.1 聊天处理 (`src/gateway/server-chat.ts`)

`createAgentEventHandler()` 处理 Agent 相关事件：

- 消息到达事件
- Agent 响应事件
- 工具调用事件
- 错误事件

### 6.2 Agent 事件文本 (`src/gateway/agent-event-assistant-text.ts`)

处理 Agent 输出的文本流事件。

### 6.3 聊天中止 (`src/gateway/chat-abort.ts`)

支持中止正在进行的 Agent 对话。

### 6.4 聊天附件 (`src/gateway/chat-attachments.ts`)

处理消息中的附件（图片、文件等）。

### 6.5 聊天清理 (`src/gateway/chat-sanitize.ts`)

清理和验证输入消息。

## 7. 节点系统

### 7.1 节点注册 (`src/gateway/node-registry.ts`)

`NodeRegistry` 管理连接到 Gateway 的节点：

- 桌面客户端节点
- 移动端节点
- 远程 Agent 节点

### 7.2 节点事件 (`src/gateway/server-node-events.ts`)

节点相关事件处理：
- 节点连接/断开
- 节点状态更新
- 节点间消息传递

### 7.3 节点订阅 (`src/gateway/server-node-subscriptions.ts`)

节点的事件订阅管理。

### 7.4 移动节点 (`src/gateway/server-mobile-nodes.ts`)

移动端节点的特殊处理逻辑。

### 7.5 节点命令策略 (`src/gateway/node-command-policy.ts`)

控制节点可执行的命令。

## 8. 插件系统

### 8.1 插件加载 (`src/gateway/server-plugins.ts`)

`loadGatewayPlugins()` 在 Gateway 启动时加载所有插件：

```
extensions/ 目录扫描
    → 读取 openclaw.plugin.json
    → 加载插件入口 (index.ts)
    → 调用 plugin.register(api)
    → 注册通道、工具、钩子
```

### 8.2 插件 HTTP 认证 (`src/gateway/server.plugin-http-auth.ts`)

为插件提供的 HTTP 端点添加认证。

## 9. 配置管理

### 9.1 配置热重载 (`src/gateway/config-reload.ts`)

`startGatewayConfigReloader()` 监控配置文件变更：

- 检测 config.yaml 变更
- 热重载配置
- 通知相关组件更新

### 9.2 运行时配置 (`src/gateway/server-runtime-config.ts`)

`resolveGatewayRuntimeConfig()` 解析 Gateway 运行时配置。

### 9.3 配置补丁 API

```
PATCH /config
// 部分更新 Gateway 配置
```

## 10. 钩子系统 (`src/gateway/hooks.ts`)

### 10.1 Gateway 钩子

```typescript
// 钩子映射 (hooks-mapping.ts)
// 将配置中的钩子映射到运行时处理器

// Gateway 支持的钩子：
// - gateway.start: Gateway 启动
// - gateway.stop: Gateway 停止
// - message.receive: 消息接收
// - message.send: 消息发送
// - agent.run.start: Agent 运行开始
// - agent.run.end: Agent 运行结束
```

## 11. 定时任务 (`src/gateway/server-cron.ts`)

`buildGatewayCronService()` 管理定时任务：

- 通道健康检查
- 缓存清理
- 统计数据收集
- 更新检查

## 12. 服务发现

### 12.1 发现运行时 (`src/gateway/server-discovery-runtime.ts`)

`startGatewayDiscovery()` 支持 Gateway 实例发现：

- mDNS 发现
- 手动注册
- 集群模式

### 12.2 发现协议 (`src/gateway/server-discovery.ts`)

定义 Gateway 实例的发现和注册协议。

## 13. Control UI

### 13.1 Web 管理界面 (`src/gateway/control-ui.ts`)

Gateway 内置 Web 管理界面：

- 实时监控仪表板
- 通道状态概览
- 会话管理
- 配置编辑
- 日志查看

### 13.2 CSP 安全 (`src/gateway/control-ui-csp.ts`)

Content Security Policy 配置，保护管理界面。

## 14. 健康检查

### 14.1 健康状态 (`src/gateway/server/health-state.ts`)

```typescript
function refreshGatewayHealthSnapshot(): HealthSnapshot
function getHealthCache(): HealthCache
function getHealthVersion(): number
function getPresenceVersion(): number
```

### 14.2 探测端点 (`src/gateway/probe.ts`)

```
GET /health
GET /probe
// 返回 Gateway 和各通道的健康状态
```

## 15. 安全

### 15.1 TLS 支持 (`src/gateway/server/tls.ts`)

`loadGatewayTlsRuntime()` 加载 TLS 证书和密钥。

### 15.2 Tailscale 集成 (`src/gateway/server-tailscale.ts`)

`startGatewayTailscaleExposure()` 通过 Tailscale 暴露 Gateway。

### 15.3 控制平面审计 (`src/gateway/control-plane-audit.ts`)

记录管理操作的审计日志。

### 15.4 控制平面速率限制 (`src/gateway/control-plane-rate-limit.ts`)

管理 API 的速率限制。

## 16. 执行审批 (`src/gateway/exec-approval-manager.ts`)

`ExecApprovalManager` 管理工具执行审批：

- Bash 命令执行前需要用户确认
- 移动端审批支持
- 自动审批规则

## 17. 部署

### 17.1 Docker 部署

```dockerfile
# Dockerfile
FROM node:22-slim
# ... 安装依赖，构建，运行 gateway
```

```yaml
# docker-compose.yml
services:
  openclaw:
    build: .
    ports:
      - "18789:18789"
    volumes:
      - config:/root/.config/openclaw
```

### 17.2 Fly.io 部署

```toml
# fly.toml
[http_service]
  internal_port = 18789
  force_https = true
```

### 17.3 启动命令

```bash
# 本地启动
openclaw gateway run --bind loopback --port 18789

# 后台启动
nohup openclaw gateway run --bind loopback --port 18789 --force > /tmp/openclaw-gateway.log 2>&1 &
```

## 18. Lane 并发管理 (`src/gateway/server-lanes.ts`)

`applyGatewayLaneConcurrency()` 配置 Agent 运行的并发限制：

- 每个 Lane (会话) 的并发运行数
- 全局最大并发运行数
- 队列溢出处理

## 19. 维护模式 (`src/gateway/server-maintenance.ts`)

`startGatewayMaintenanceTimers()` 管理 Gateway 维护任务：

- 内存使用监控 (`server-startup-memory.ts`)
- 重启哨兵 (`server-restart-sentinel.ts`)
- 优雅关闭 (`server-close.ts`)

## 关键源码文件索引

| 文件路径 | 说明 |
|---------|------|
| `src/gateway/server.ts` | Gateway 导出入口 |
| `src/gateway/server.impl.ts` | Gateway 实现主体 |
| `src/gateway/server-http.ts` | HTTP 服务器 |
| `src/gateway/server-ws-runtime.ts` | WebSocket 运行时 |
| `src/gateway/server-channels.ts` | 通道管理器 |
| `src/gateway/server-chat.ts` | Agent 聊天处理 |
| `src/gateway/server-methods.ts` | API 方法处理 |
| `src/gateway/server-methods-list.ts` | 方法列表 |
| `src/gateway/server-plugins.ts` | 插件加载 |
| `src/gateway/server-cron.ts` | 定时任务 |
| `src/gateway/server-discovery-runtime.ts` | 服务发现 |
| `src/gateway/auth.ts` | 认证 |
| `src/gateway/auth-rate-limit.ts` | 认证速率限制 |
| `src/gateway/config-reload.ts` | 配置热重载 |
| `src/gateway/hooks.ts` | 钩子系统 |
| `src/gateway/node-registry.ts` | 节点注册 |
| `src/gateway/control-ui.ts` | 管理界面 |
| `src/gateway/openai-http.ts` | OpenAI 兼容 API |
| `src/gateway/exec-approval-manager.ts` | 执行审批 |
| `src/gateway/server-lanes.ts` | 并发管理 |
| `src/gateway/server-close.ts` | 优雅关闭 |
| `src/gateway/server/health-state.ts` | 健康状态 |
| `src/gateway/server/tls.ts` | TLS 支持 |
