# OpenClaw 架构文档体系

本目录包含 OpenClaw 项目各核心子系统的底层运行机制详细说明文档。

## 文档索引

| 文档 | 描述 |
|------|------|
| [01-agent-runtime.md](./01-agent-runtime.md) | Agent 运行时机制 — 启动流程、生命周期、进程模型 |
| [02-message-response.md](./02-message-response.md) | 消息响应回复机制 — 消息接收、路由、回复流程 |
| [03-skill-mechanism.md](./03-skill-mechanism.md) | Skill 机制 — 技能定义、加载、执行管线 |
| [04-context-management.md](./04-context-management.md) | Agent 上下文管理 — 上下文窗口、压缩、记忆系统 |
| [05-session-management.md](./05-session-management.md) | 会话管理 — 会话生命周期、持久化、多用户隔离 |
| [06-feishu-plugin.md](./06-feishu-plugin.md) | 飞书插件 — Feishu/Lark 集成架构与消息处理 |
| [07-gateway.md](./07-gateway.md) | OpenClaw Gateway — 网关架构、API、通道桥接 |

## 项目总体架构概览

```
openclaw/
├── src/                    # 核心源码 (TypeScript ESM)
│   ├── agents/             # Agent 核心引擎 (嵌入式 Pi Agent 运行器)
│   ├── gateway/            # Gateway 服务器 (HTTP/WS API)
│   ├── channels/           # 通道抽象层 (插件注册表)
│   ├── routing/            # 消息路由 (绑定匹配)
│   ├── sessions/           # 会话状态管理
│   ├── auto-reply/         # 自动回复与模板引擎
│   ├── providers/          # LLM 供应商集成
│   ├── plugins/            # 插件注册与钩子系统
│   ├── config/             # 配置加载与管理
│   ├── memory/             # 内存/记忆子系统
│   ├── process/            # 进程管理与命令队列
│   └── hooks/              # 钩子系统
├── extensions/             # 通道插件 (Feishu, Discord, Telegram 等)
├── skills/                 # 内置技能包
├── packages/               # 独立包 (clawdbot, moltbot)
├── apps/                   # 原生应用 (macOS, iOS, Android)
├── ui/                     # Web UI 组件
└── docs/                   # 文档
```

## 核心设计原则

1. **多通道统一网关**: Gateway 作为中心枢纽，桥接所有消息通道与 AI Agent
2. **插件化架构**: 通道、技能、内存系统均通过插件 API 扩展
3. **嵌入式 Agent 运行器**: 基于 `pi-coding-agent` 库，支持多供应商 LLM
4. **绑定路由系统**: 通过配置绑定 (bindings) 将不同通道/用户映射到不同 Agent
5. **会话隔离**: 每个通道+用户+Agent 组合维护独立的会话上下文
