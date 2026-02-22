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

`loadSkillEntries()` 从 6 个层级源加载 Skill，优先级从低到高：

```
Extra Dirs < Bundled < Managed < Agents-Personal < Agents-Project < Workspace
```

**源路径**：
1. **Extra Dirs**: 用户配置路径 (`config.skills.load.extraDirs`)
2. **Bundled**: `resolveBundledSkillsDir()` — 随 OpenClaw 安装
3. **Managed**: `{CONFIG_DIR}/skills` (如 `~/.config/openclaw/skills`)
4. **Personal Agents**: `~/.agents/skills`
5. **Project Agents**: `{workspaceDir}/.agents/skills`
6. **Workspace**: `{workspaceDir}/skills`

后加载的源通过 `Map<name, Skill>` 合并覆盖先加载的。

**每个源目录的发现流程**：
1. `listChildDirectories()` 枚举子目录（跳过隐藏目录和 `node_modules`）
2. 可疑检查: 超过 `maxCandidatesPerRoot` (默认 300) 时警告并截断
3. 对每个子目录（上限 `maxSkillsLoadedPerSource`，默认 200）：
   - 验证 `{child}/SKILL.md` 存在
   - 大小检查: 拒绝超过 256KB 的 SKILL.md
   - 通过 `loadSkillsFromDir()` 加载
4. `resolveOpenClawMetadata()` 提取 OpenClaw 特有配置
5. `compactSkillPaths()` 将 home 路径替换为 `~` 节省约 5-6 tokens/skill

**关键常量**：

```typescript
const DEFAULT_MAX_CANDIDATES_PER_ROOT = 300;
const DEFAULT_MAX_SKILLS_LOADED_PER_SOURCE = 200;
const DEFAULT_MAX_SKILLS_IN_PROMPT = 150;
const DEFAULT_MAX_SKILLS_PROMPT_CHARS = 30_000;
const DEFAULT_MAX_SKILL_FILE_BYTES = 256_000;
```

### 3.3 Skill 过滤 (`src/agents/skills/filter.ts`, `config.ts`)

加载后的 Skill 经过多层过滤：

**`shouldIncludeSkill()` 决策树**（按顺序）：

1. **配置禁用**: `skillConfig.enabled === false` → 拒绝
2. **内置黑名单**: 不在 `allowBundled` 允许列表 → 拒绝
3. **平台不匹配**: Skill 声明的 `os: [...]` 与当前平台不兼容 → 拒绝
4. **始终加载覆盖**: `metadata.always === true` → 接受（跳过后续检查）
5. **运行时依赖检查** (`evaluateRuntimeRequires()`):
   - `requires.bins`: 所有二进制必须存在
   - `requires.anyBins`: 至少一个二进制存在
   - `requires.env`: 所有环境变量必须设置
   - `requires.config`: 配置路径必须有效

**过滤器规范化**：

```typescript
function normalizeSkillFilter(skillFilter?: ReadonlyArray<unknown>): string[] | undefined
// 输入: ["github", "  ", "git", null, ""]
// 输出: ["github", "git"]
```

### 3.4 提示词限制（二分搜索）

`applySkillsPromptLimits()` 两阶段截断：

1. **数量限制**: 最多 `maxSkillsInPrompt` (默认 150) 个 Skill
2. **字符限制**: 若仍超过 `maxSkillsPromptChars` (默认 30KB)，使用二分搜索找到最大可容纳的前缀

```typescript
let lo = 0, hi = skillsForPrompt.length;
while (lo < hi) {
  const mid = Math.ceil((lo + hi) / 2);
  if (fits(skillsForPrompt.slice(0, mid))) lo = mid;
  else hi = mid - 1;
}
```

### 3.5 Skill 同步

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

**命令名卫生化**：

```typescript
function sanitizeSkillCommandName(raw: string): string
// "My-API@Tool v2" → "my_api_tool_v2"
// 规则: 小写 → 非字母数字替换为 _ → 合并连续 _ → 去首尾 _ → 截断到 32 字符
```

**命令名去重**：若命令名冲突，自动添加 `_2`, `_3` 等后缀（最多尝试到 `_999`）。

**描述截断**: 最多 100 字符（Discord 限制）。

### 6.3 命令执行

用户输入 `/command args` 时：
1. 匹配命令名到对应 Skill
2. 如有 `dispatch` 配置，直接调用指定工具
3. 否则，读取 SKILL.md 并让 Agent 按说明处理

## 7. Skill 安装系统

### 7.1 安装管线 (`src/agents/skills-install.ts`)

Skill 可声明安装依赖，系统自动安装：

```typescript
type SkillInstallRequest = {
  workspaceDir: string;
  skillName: string;
  installId: string;           // resolveInstallId(spec, index) → "{kind}-{index}"
  timeoutMs?: number;          // 限制 1-900 秒
  config?: OpenClawConfig;
};

type SkillInstallResult = {
  ok: boolean;
  message: string;
  stdout: string;
  stderr: string;
  code: number | null;
  warnings?: string[];         // 安全扫描警告
};
```

### 7.2 各安装类型的命令生成

| 类型 | 命令 | 说明 |
|------|------|------|
| **brew** | `brew install {formula}` | Homebrew 安装 |
| **node** | `{npm\|pnpm\|yarn\|bun} install -g --ignore-scripts {package}` | Node 全局安装，强制 `--ignore-scripts` |
| **go** | `go install {module}` | Go 工具安装 |
| **uv** | `uv tool install {package}` | Python UV 安装 |
| **download** | 自定义下载 + 解压 | 支持 tar.gz/tar.bz2/zip |

### 7.3 前置依赖自动安装

安装前自动检查所需工具链是否存在：

- **UV**: 若 `uv` 不在 PATH，尝试 `brew install uv`
- **Go**: 若 `go` 不在 PATH，尝试 `brew install go` 或 `apt-get install golang-go`
- **Brew 二进制目录**: `resolveBrewBinDir()` 依次尝试 `brew --prefix`、`$HOMEBREW_PREFIX`、回退路径

### 7.4 安全扫描

```typescript
async function collectSkillInstallScanWarnings(entry: SkillEntry): Promise<string[]>
```

安装前通过 `scanDirectoryWithSummary()` 扫描 Skill 目录：
- 危险代码模式 → WARNING
- 可疑代码模式 → 提示运行审计
- 扫描失败 → 记录但继续安装

### 7.5 安装偏好

```typescript
function resolveSkillsInstallPreferences(config?: OpenClawConfig): {
  preferBrew: boolean;            // 优先使用 Homebrew
  nodeManager: "npm" | "pnpm" | "yarn" | "bun";  // Node 包管理器
}
```

## 8. 内置 Skill 列表

项目根目录 `skills/` 包含 52 个内置 Skill：

| 分类 | Skill 列表 |
|------|-----------|
| **生产力** | `apple-notes`, `apple-reminders`, `bear-notes`, `notion`, `obsidian`, `things-mac`, `trello` |
| **通讯** | `slack`, `discord`, `imsg`, `bluebubbles`, `wacli` (WhatsApp), `himalaya` (email) |
| **开发** | `github`, `gh-issues`, `tmux`, `coding-agent`, `clawhub` |
| **媒体** | `spotify-player`, `songsee`, `sonoscli`, `openai-image-gen`, `video-frames`, `gifgrep`, `camsnap`, `peekaboo`, `canvas` |
| **AI/语音** | `gemini`, `openai-whisper`, `openai-whisper-api`, `sherpa-onnx-tts`, `voice-call` |
| **文档** | `nano-pdf`, `summarize`, `blogwatcher` |
| **系统** | `weather`, `healthcheck`, `session-logs`, `model-usage`, `skill-creator` |
| **工具** | `food-order`, `goplaces`, `oracle`, `ordercli`, `sag`, `sherpa-onnx-tts` |
| **智能家居** | `openhue` |
| **其他** | `1password`, `blucli`, `eightctl`, `gog`, `mcporter`, `nano-banana-pro` |

### 扩展 Skill

扩展也可以提供 Skill：

| 扩展 | Skill 目录 | 内容 |
|------|-----------|------|
| `extensions/feishu/skills/` | 飞书特有技能 | `feishu-doc`, `feishu-drive`, `feishu-perm`, `feishu-wiki` |
| `extensions/open-prose/skills/` | 文本编辑技能 | `prose` — OpenProse VM 技能包 |

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

Skill 可通过 `applySkillEnvOverrides()` 在运行时覆盖环境变量：

**安全硬性阻止列表**：

```typescript
const HARD_BLOCKED_SKILL_ENV_PATTERNS: ReadonlyArray<RegExp> = [
  /^NODE_OPTIONS$/i,
  /^OPENSSL_CONF$/i,
  /^LD_PRELOAD$/i,
  /^DYLD_INSERT_LIBRARIES$/i,
];
```

阻止原因: 防止加载器操纵注入代码。

**覆盖流程**：
1. 遍历所有 Skill 的配置 (`config.skills.entries[skillKey]`)
2. 对每个 Skill 的 `env` 字段进行卫生化 (`sanitizeSkillEnvOverrides`)
3. 仅允许 `requires.env` 和 `primaryEnv` 中声明的敏感变量
4. 验证值（检查 null 字节等）
5. 返回环境恢复器函数，运行结束后恢复 `process.env`

**配置来源**：
```typescript
// config.skills.entries[skillKey]
{
  env?: Record<string, string>;   // 键值覆盖
  apiKey?: string;                 // 填充到 primaryEnv
}
```

## 10. Skill 刷新机制

### 10.1 文件监听 (`src/agents/skills/refresh.ts`)

基于 `chokidar` 的文件监听，支持 Gateway 运行时热重载。

**监听路径**：
- `{workspaceDir}/skills/`
- `{workspaceDir}/.agents/skills/`
- `{CONFIG_DIR}/skills/`
- `~/.agents/skills/`
- 额外配置目录和插件 Skill 目录

**监听目标** (Glob 模式)：
- `{root}/SKILL.md` — 根级 Skill
- `{root}/*/SKILL.md` — 子目录 Skill

**忽略模式**：

```typescript
const DEFAULT_SKILLS_WATCH_IGNORED: RegExp[] = [
  /(^|[\\/])\.git([\\/]|$)/,
  /(^|[\\/])node_modules([\\/]|$)/,
  /(^|[\\/])dist([\\/]|$)/,
  /(^|[\\/])\.venv([\\/]|$)/,
  /(^|[\\/])__pycache__([\\/]|$)/,
  // ... 更多构建目录
];
```

### 10.2 版本跟踪与热重载

```typescript
const workspaceVersions = new Map<string, number>();
let globalVersion = 0;

type SkillsChangeEvent = {
  workspaceDir?: string;
  reason: "watch" | "manual" | "remote-node";
  changedPath?: string;
};

function registerSkillsChangeListener(
  listener: (event: SkillsChangeEvent) => void
): () => void  // 返回注销函数
```

文件变更通过 250ms 防抖后触发 `bumpSkillsSnapshotVersion()`，通知所有注册的监听器重新加载 Skill 列表。

## 11. 插件 Skill 发现 (`src/agents/skills/plugin-skills.ts`)

`resolvePluginSkillDirs()` 从插件清单注册表中发现 Skill 目录：

1. 加载插件清单注册表 (`loadPluginManifestRegistry()`)
2. 对每个声明了 `skills` 的插件：
   - 检查启用状态 (`resolveEnableState`)
   - 检查记忆槽位决策 (`resolveMemorySlotDecision`) — 同一槽位仅允许一个记忆插件
3. 去重并验证路径存在

## 12. 实际 SKILL.md 示例

### 示例 1: openai-image-gen (带安装依赖)

```yaml
---
name: openai-image-gen
description: Batch-generate images via OpenAI Images API.
metadata:
  openclaw:
    emoji: "🖼️"
    requires: { bins: ["python3"], env: ["OPENAI_API_KEY"] }
    primaryEnv: OPENAI_API_KEY
    install:
      - id: python-brew
        kind: brew
        formula: python
        bins: ["python3"]
        label: "Install Python (brew)"
---
```

### 示例 2: spotify-player (anyBins 条件)

```yaml
---
name: spotify-player
description: Terminal Spotify playback/search via spogo or spotify_player.
metadata:
  openclaw:
    emoji: "🎵"
    requires: { anyBins: ["spogo", "spotify_player"] }
    install:
      - id: brew
        kind: brew
        formula: spogo
        bins: ["spogo"]
      - id: brew
        kind: brew
        formula: spotify_player
        bins: ["spotify_player"]
---
```

### 示例 3: prose (纯文档型)

```yaml
---
name: prose
description: OpenProse VM skill pack.
metadata:
  openclaw: { emoji: "🪶", homepage: "https://www.prose.md" }
---
```

## 13. Gateway Skill API (`src/gateway/server-methods/skills.ts`)

Gateway 暴露 4 个 Skill 相关的 JSON-RPC 方法：

| 方法 | 说明 |
|------|------|
| `skills.status` | 返回工作区 Skill 状态报告（已加载、缺失依赖等） |
| `skills.bins` | 收集所有 Skill 声明的二进制依赖列表 |
| `skills.install` | 安装指定 Skill 的依赖 |
| `skills.update` | 更新 Skill 配置（API Key、环境变量等） |

## 14. CLI 命令 (`src/cli/skills-cli.ts`)

```bash
openclaw skills list              # 列出所有 Skill
openclaw skills list --eligible   # 仅显示满足条件的 Skill
openclaw skills list --json       # JSON 格式输出
openclaw skills list -v           # 显示缺失的依赖
openclaw skills info <name>       # 查看指定 Skill 详情
openclaw skills check             # 检查所有 Skill 的依赖状态
```

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
| `src/gateway/server-methods/skills.ts` | Gateway Skill API 端点 |
| `src/cli/skills-cli.ts` | CLI skills 子命令 |
| `src/auto-reply/skill-commands.ts` | 斜杠命令解析与派发 |
| `skills/` | 内置 Skill 目录 (52 个) |
| `extensions/*/skills/` | 扩展 Skill 目录 |
