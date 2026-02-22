# OpenClaw Skill 机制

## 概述

OpenClaw 的 Skill（技能）机制为 Agent 提供可扩展的能力模块。每个 Skill 是一个独立的知识/行为包，包含 Markdown 格式的技能说明文件 (SKILL.md)，Agent 在运行时根据用户请求动态选择并加载相应的 Skill。

## 1. Skill 定义格式

### 1.1 目录结构

每个 Skill 是一个目录，必须包含一个 `SKILL.md` 文件：

```
skills/
├── weather/
│   └── SKILL.md           # 技能说明
├── slack/
│   └── SKILL.md
├── notion/
│   ├── SKILL.md
│   └── scripts/           # 可选的辅助脚本
└── nano-pdf/
    └── SKILL.md
```

### 1.2 SKILL.md 格式

SKILL.md 使用 Markdown 编写，支持 YAML frontmatter 头部：

```markdown
---
name: Weather
description: Get weather forecasts and conditions
openclaw:
  skillKey: weather
  primaryEnv: OPENWEATHER_API_KEY
  emoji: "🌤"
  os: ["linux", "darwin"]
  requires:
    bins: ["curl"]
    env: ["OPENWEATHER_API_KEY"]
invocation:
  user-invocable: true
  disable-model-invocation: false
commands:
  - name: weather
    description: Check weather for a location
---

# Weather Skill

Instructions for the agent on how to check weather...
```

### 1.3 Frontmatter 字段详解

**OpenClaw 元数据** (`openclaw`):

| 字段 | 类型 | 说明 |
|------|------|------|
| `skillKey` | string | 技能唯一标识符 |
| `primaryEnv` | string | 主要环境变量名 |
| `emoji` | string | 显示用表情 |
| `always` | boolean | 是否总是加载 (不可过滤) |
| `os` | string[] | 支持的操作系统 |
| `homepage` | string | 技能主页 URL |
| `requires.bins` | string[] | 必须存在的二进制依赖 |
| `requires.anyBins` | string[] | 至少存在一个的二进制依赖 |
| `requires.env` | string[] | 必须设置的环境变量 |
| `requires.config` | string[] | 必须存在的配置路径 |
| `install` | SkillInstallSpec[] | 安装规格 |

**调用策略** (`invocation`):

| 字段 | 类型 | 说明 |
|------|------|------|
| `user-invocable` | boolean | 是否可由用户通过 /command 触发 |
| `disable-model-invocation` | boolean | 是否禁止 LLM 自动触发 |

**命令定义** (`commands`):

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 命令名称 (用户输入 /name 触发) |
| `description` | string | 命令描述 |
| `dispatch.kind` | string | 派发类型 ("tool") |
| `dispatch.toolName` | string | 绑定的工具名称 |

## 2. Skill 类型定义 (`src/agents/skills/types.ts`)

```typescript
type SkillEntry = {
  skill: Skill;                         // 底层技能对象
  frontmatter: ParsedSkillFrontmatter;  // 解析后的 frontmatter
  metadata?: OpenClawSkillMetadata;     // OpenClaw 特有元数据
  invocation?: SkillInvocationPolicy;   // 调用策略
};

type SkillSnapshot = {
  entries: SkillEntry[];
  prompt: string;                       // 生成的提示词文本
};

type SkillInstallSpec = {
  kind: "brew" | "node" | "go" | "uv" | "download";
  label?: string;
  bins?: string[];
  os?: string[];
  formula?: string;      // brew formula
  package?: string;      // node/go/uv package
  url?: string;          // download URL
  archive?: string;      // 归档类型
  extract?: boolean;
  targetDir?: string;
};

type SkillEligibilityContext = {
  remote?: {
    platforms: string[];
    hasBin: (bin: string) => boolean;
    hasAnyBin: (bins: string[]) => boolean;
  };
};
```

## 3. Skill 加载流程

### 3.1 加载源

Skill 从多个目录加载并合并：

```
                     ┌─────────────────┐
                     │ loadWorkspace    │
                     │ SkillEntries()   │
                     └────────┬────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                  ▼
   ┌────────────────┐ ┌──────────────┐ ┌────────────────┐
   │ 内置 Skills    │ │ 插件 Skills  │ │ 工作区 Skills  │
   │ skills/        │ │ extensions/  │ │ .agent/skills/ │
   │                │ │ */skills/    │ │ (用户自定义)   │
   └────────────────┘ └──────────────┘ └────────────────┘
```

### 3.2 加载步骤 (`src/agents/skills/workspace.ts`)

1. **发现**: `loadWorkspaceSkillEntries()` 扫描所有 Skill 目录
2. **解析**: `loadSkillsFromDir()` (from `pi-coding-agent`) 读取每个 SKILL.md
3. **Frontmatter 解析**: `parseFrontmatter()` 提取 YAML 头部
4. **元数据解析**: `resolveOpenClawMetadata()` 提取 OpenClaw 特有配置
5. **路径压缩**: `compactSkillPaths()` 将 home 路径替换为 `~` 节省 token

### 3.3 Skill 过滤

加载后的 Skill 经过多层过滤：

```typescript
function filterSkillEntries(
  entries: SkillEntry[],
  config?: OpenClawConfig,
  skillFilter?: string[],
  eligibility?: SkillEligibilityContext,
): SkillEntry[]
```

过滤条件：
- `shouldIncludeSkill()`: 检查二进制依赖、环境变量、配置路径
- 操作系统兼容性检查
- 用户配置的 Skill 过滤器
- 内置允许列表 (`resolveBundledAllowlist`)

### 3.4 Skill 同步

`syncSkillsToWorkspace()` 将远程/管理的 Skill 同步到工作区目录。

## 4. Skill 注入到系统提示词

### 4.1 提示词构建 (`buildWorkspaceSkillsPrompt`)

```typescript
function buildWorkspaceSkillsPrompt(params: {
  config?: OpenClawConfig;
  skillFilter?: string[];
  eligibility?: SkillEligibilityContext;
}): string
```

生成的 Skill 提示词格式：

```
<available_skills>
  <skill name="weather" location="~/skills/weather/SKILL.md">
    <description>Get weather forecasts and conditions</description>
  </skill>
  <skill name="slack" location="~/skills/slack/SKILL.md">
    <description>Interact with Slack workspaces</description>
  </skill>
</available_skills>
```

### 4.2 系统提示词中的 Skill 部分 (`system-prompt.ts`)

```
## Skills (mandatory)
Before replying: scan <available_skills> <description> entries.
- If exactly one skill clearly applies: read its SKILL.md at <location> with `read`, then follow it.
- If multiple could apply: choose the most specific one, then read/follow it.
- If none clearly apply: do not read any SKILL.md.
Constraints: never read more than one skill up front; only read after selecting.

<available_skills>
  ...
</available_skills>
```

关键设计：Agent 不会预加载所有 Skill 内容，而是根据请求按需用 read 工具读取对应的 SKILL.md。

## 5. Skill 运行时解析 (`resolveSkillsPromptForRun`)

每次 Agent 运行前都会解析当前可用的 Skill：

```typescript
function resolveSkillsPromptForRun(params: {
  config?: OpenClawConfig;
  sessionKey?: string;
  skillFilter?: string[];
}): string
```

该函数：
1. 加载工作区 Skill 条目
2. 应用过滤器
3. 构建快照 (`SkillSnapshot`)
4. 生成提示词文本

## 6. Skill 命令系统

### 6.1 用户可调用命令

部分 Skill 注册为用户可调用命令（斜杠命令）：

```typescript
type SkillCommandSpec = {
  name: string;              // /命令名
  skillName: string;         // 所属 Skill
  description: string;       // 描述
  dispatch?: {
    kind: "tool";
    toolName: string;        // 直接调用的工具
    argMode?: "raw";         // 参数转发模式
  };
};
```

### 6.2 命令构建 (`buildWorkspaceSkillCommandSpecs`)

从 Skill frontmatter 中提取命令定义，生成命令规格列表。

### 6.3 命令执行

用户输入 `/command args` 时：
1. 匹配命令名到对应 Skill
2. 如有 `dispatch` 配置，直接调用指定工具
3. 否则，读取 SKILL.md 并让 Agent 按说明处理

## 7. Skill 安装系统

### 7.1 安装管线 (`src/agents/skills-install.ts`)

Skill 可声明安装依赖，系统自动安装：

```typescript
type SkillInstallSpec = {
  kind: "brew" | "node" | "go" | "uv" | "download";
  // brew: 使用 Homebrew 安装
  // node: 使用 npm/pnpm/yarn/bun 安装
  // go: 使用 go install 安装
  // uv: 使用 uv (Python) 安装
  // download: 直接下载二进制
};
```

### 7.2 安装偏好 (`resolveSkillsInstallPreferences`)

```typescript
function resolveSkillsInstallPreferences(config?: OpenClawConfig): {
  preferBrew: boolean;            // 优先使用 Homebrew
  nodeManager: "npm" | "pnpm" | "yarn" | "bun";  // Node 包管理器
}
```

### 7.3 安装流程

1. 检查 Skill 声明的依赖是否已安装
2. 按 `SkillInstallSpec` 中的规格执行安装
3. 支持下载 tar.bz2 归档 (`skills-install-download.ts`)
4. 安装状态跟踪 (`skills-status.ts`)

## 8. 内置 Skill 列表

项目根目录 `skills/` 包含以下内置 Skill：

| Skill | 描述 |
|-------|------|
| `weather` | 天气查询 |
| `slack` | Slack 工作区交互 |
| `notion` | Notion 集成 |
| `apple-reminders` | Apple 提醒事项 |
| `things-mac` | Things (macOS) 任务管理 |
| `spotify-player` | Spotify 播放控制 |
| `clawhub` | ClawHub 集成 |
| `nano-pdf` | PDF 处理 |
| `nano-banana-pro` | Banana Pro 集成 |
| `summarize` | 内容摘要 |
| `gifgrep` | GIF 搜索 |
| `openai-image-gen` | OpenAI 图像生成 |

### 扩展 Skill

扩展也可以提供 Skill：

| 扩展 | Skill 目录 |
|------|-----------|
| `extensions/feishu/skills/` | 飞书特有技能 |
| `extensions/open-prose/skills/` | 文本编辑技能 |

## 9. Skill 配置 (`src/agents/skills/config.ts`)

### 9.1 配置检查

```typescript
function shouldIncludeSkill(params: {
  entry: SkillEntry;
  config?: OpenClawConfig;
  eligibility?: SkillEligibilityContext;
}): boolean
```

检查：
- `hasBinary()`: 验证二进制依赖存在
- `isConfigPathTruthy()`: 验证配置路径有效
- `isBundledSkillAllowed()`: 检查内置 Skill 允许列表
- `resolveRuntimePlatform()`: 检查平台兼容性

### 9.2 环境覆盖 (`src/agents/skills/env-overrides.ts`)

Skill 可通过 `applySkillEnvOverrides()` 在运行时覆盖环境变量。

## 10. Skill 刷新机制

### 10.1 监听变更 (`src/agents/skills/refresh.ts`)

`registerSkillsChangeListener()` 注册文件变更监听器：

- 监控 Skill 目录的文件变更
- 变更时自动重新加载 Skill 列表
- Gateway 运行时热重载

## 关键源码文件索引

| 文件路径 | 说明 |
|---------|------|
| `src/agents/skills.ts` | Skill 模块入口与导出 |
| `src/agents/skills/types.ts` | 核心类型定义 |
| `src/agents/skills/workspace.ts` | 工作区 Skill 加载与构建 |
| `src/agents/skills/config.ts` | Skill 配置与过滤 |
| `src/agents/skills/frontmatter.ts` | Frontmatter 解析 |
| `src/agents/skills/env-overrides.ts` | 环境变量覆盖 |
| `src/agents/skills/filter.ts` | Skill 过滤器 |
| `src/agents/skills/plugin-skills.ts` | 插件 Skill 发现 |
| `src/agents/skills/refresh.ts` | Skill 热重载 |
| `src/agents/skills-install.ts` | 安装系统 |
| `src/agents/skills-status.ts` | 安装状态 |
| `src/agents/system-prompt.ts` | 系统提示词中的 Skill 部分 |
| `skills/` | 内置 Skill 目录 |
