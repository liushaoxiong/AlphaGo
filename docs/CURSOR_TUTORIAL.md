# Cursor IDE 完全使用教程（2026 最新版）

> **文档版本**：基于 Cursor 3.9（2026-06-22）  
> **最后更新**：2026-06-29  
> **官方文档**：[cursor.com/docs](https://cursor.com/docs) · [中文文档](https://cursor.com/cn/docs)

---

## 目录

1. [产品概述](#1-产品概述)
2. [安装与快速上手](#2-安装与快速上手)
3. [核心功能详解](#3-核心功能详解)
4. [AI 交互模式](#4-ai-交互模式)
5. [上下文与 @ 提及](#5-上下文与--提及)
6. [Rules、Skills 与 Commands](#6-rulesskills-与-commands)
7. [MCP 外部工具集成](#7-mcp-外部工具集成)
8. [Agents Window 与 Cloud Agents](#8-agents-window-与-cloud-agents)
9. [终端集成与安全策略](#9-终端集成与安全策略)
10. [Bugbot 代码审查](#10-bugbot-代码审查)
11. [Customize 与 Plugins（3.9）](#11-customize-与-plugins39)
12. [典型使用案例](#12-典型使用案例)
13. [快捷键速查表](#13-快捷键速查表)
14. [最佳实践](#14-最佳实践)
15. [定价方案](#15-定价方案)
16. [常见问题](#16-常见问题)
17. [官方资源](#17-官方资源)

---

## 1. 产品概述

Cursor 是由 Anysphere 开发的 **AI 原生 IDE**，基于 VS Code 开源代码库构建。它不只是「带聊天框的编辑器」，而是一套完整的 **编码 Agent（Coding Agent）** 平台，能够：

- 理解整个代码库的语义结构
- 跨多个文件自主编辑代码
- 在终端中运行命令、安装依赖、执行测试
- 通过浏览器打开页面并验证 UI
- 在云端 VM 中并行执行长时间任务
- 与 GitHub、GitLab、Jira、Notion 等工具深度集成

### 2026 年重要版本里程碑

| 时间 | 版本 | 重要更新 |
|------|------|----------|
| 2026-04-02 | Cursor 3 | **Agents Window**（Agent 优先工作区）正式发布 |
| 2026-05-18 | — | **Composer 2.5** 自研模型发布 |
| 2026-05-22 | 3.5 | 终端执行模式重构 |
| 2026-05-29 | 3.6 | **Auto-review** 成为推荐默认执行模式 |
| 2026-06-17 | 3.7 | 云环境配置、Cloud Subagents |
| 2026-06-18 | 3.8 | **Automations** 改进 |
| 2026-06-22 | 3.9 | **Customize** 统一配置页、Plugins 体系 |

### 适用平台

- macOS 12+
- Windows 10+
- Linux（Debian/Ubuntu、RHEL/Fedora）

---

## 2. 安装与快速上手

### 2.1 安装

1. 访问 [cursor.com](https://cursor.com) 下载对应平台的安装包
2. 安装后启动，使用 GitHub 或邮箱登录
3. 选择订阅计划（免费 Hobby 计划即可开始体验）

### 2.2 从 VS Code 迁移

Cursor 兼容 VS Code 扩展和设置。首次启动时可选择导入 VS Code 配置，包括：

- 已安装的扩展
- 键盘快捷键
- 主题与外观设置
- `settings.json` 配置

### 2.3 五分钟快速上手

```
步骤 1：File → Open Folder，打开你的项目
步骤 2：按 Cmd/Ctrl + I 打开 Agent 面板
步骤 3：输入：「解释这个代码库的结构和入口点」
步骤 4：选一个安全的小改动让 Agent 实施，在 diff 视图中审查
步骤 5：让 Agent 运行项目的 lint/test 命令验证结果
```

---

## 3. 核心功能详解

### 3.1 功能全景图

```
┌─────────────────────────────────────────────────────────┐
│                    Cursor IDE 功能架构                    │
├─────────────┬──────────────┬──────────────┬──────────────┤
│  编码辅助    │   AI Agent   │   团队协作    │   扩展生态    │
├─────────────┼──────────────┼──────────────┼──────────────┤
│ Tab 补全     │ Agent 模式   │ Cloud Agents │ MCP 集成     │
│ Inline Edit  │ Ask 模式     │ Bugbot       │ Plugins      │
│ 跨文件预测   │ Plan 模式    │ 团队 Rules   │ Skills       │
│              │ Debug 模式   │ PR 管理      │ Hooks        │
└─────────────┴──────────────┴──────────────┴──────────────┘
```

### 3.2 Tab 自动补全（Copilot++）

Tab 是 Cursor 的 **AI 驱动自动补全**，比传统单行补全更智能。

**核心能力：**

- 多行代码补全，自动添加 import 语句
- 根据最近编辑、周围代码、Linter 错误给出上下文感知建议
- **Jump-in-file**：接受建议后再按 Tab，跳转到预测的下一处编辑位置
- **跨文件编辑**：预测到其他文件需要同步修改时，底部会出现 Portal 窗口

**操作方式：**

| 操作 | Mac | Windows/Linux |
|------|-----|---------------|
| 接受完整建议 | `Tab` | `Tab` |
| 逐词接受 | `Cmd + →` | `Ctrl + →` |
| 拒绝建议 | `Esc` 或继续输入 | 同左 |

**开关控制**：点击右下角 Tab 状态指示器，可 Snooze（暂停）、全局禁用或按文件扩展名禁用。

> **注意**：Rules 不作用于 Tab 补全，只作用于 Agent/Chat。

### 3.3 Inline Edit（行内编辑）

适合对选中代码做精准、局部的修改，无需打开 Agent 面板。

**使用流程：**

1. 选中要修改的代码
2. 按 `Cmd/Ctrl + K`
3. 用自然语言描述修改意图，例如：「把这个改成 async/await，并加上错误处理」
4. 按 `Return` 查看 diff
5. 按 `Cmd/Ctrl + Enter` 接受修改
6. 追问细节：按 `Opt/Alt + Return` 进入问答模式

### 3.4 多文件编辑

| 方式 | 快捷键 | 适用场景 |
|------|--------|----------|
| **Inline Edit** | `Cmd/Ctrl + K` | 选中代码的局部修改 |
| **Agent** | `Cmd/Ctrl + I` | 跨文件、需跑命令的复杂任务 |
| **Tab 跨文件** | 自动 Portal | 关联文件的同步小改动 |
| **Plan Mode** | `Shift+Tab` 切换 | 大改动先审方案再编码 |

### 3.5 Composer 2.5 模型

Composer 2.5 是 Cursor 自研的 Agent 模型，针对短回合、工具调用、迭代编辑优化：

- 输入：$0.5 / 百万 tokens
- 输出：$2.5 / 百万 tokens
- 在 Auto 模式下自动路由，成本低于第三方顶级模型
- 适合日常编码、快速迭代场景

---

## 4. AI 交互模式

按 `Shift + Tab` 循环切换模式，或点击模式选择器。

### 4.1 模式对比

| 模式 | 用途 | 能否改代码 | 典型场景 |
|------|------|------------|----------|
| **Agent** | 自主完成编码任务 | ✅ | 新功能、重构、修 Bug、写测试 |
| **Ask** | 只读问答 | ❌ | 理解架构、解释函数、探索代码 |
| **Plan** | 先出方案再编码 | ✅（批准后） | 跨多文件、需求不清晰的复杂功能 |
| **Debug** | 基于运行时证据排错 | ✅ | 难复现 Bug、需日志/堆栈分析 |

### 4.2 Agent 模式

Agent 是 Cursor 的核心工作流，支持长循环：读代码 → 改文件 → 跑命令 → 迭代。

**Agent 能做什么：**

- 搜索代码库、读写多个文件
- 自动运行终端命令（安装依赖、跑测试、构建）
- 使用浏览器打开页面、点击、填表、截图
- 生成图片（UI 原型、架构图）
- 委派 **子代理（Subagents）** 并行处理调研、Shell、浏览器任务
- 支持消息队列：Agent 工作时可排队发送后续指令
- **Restore Checkpoint**：回滚到某条消息之前的状态

**打开方式：**

- `Cmd/Ctrl + I`：打开 Agent 侧栏
- `Cmd/Ctrl + L`：打开 Chat 侧栏
- 选中代码后按 `Cmd/Ctrl + L`：带着选中内容打开 Agent

### 4.3 Ask 模式

只读模式，Agent 不会修改任何文件，适合学习和探索。

**示例提示词：**

```
认证流程是怎么工作的？从用户登录到 token 刷新的完整链路是什么？
请结合 @src/auth/ 目录下的代码解释。
```

### 4.4 Plan 模式

适合需求不清晰或改动范围大的复杂功能，Agent 会先输出实现计划供你审查。

**使用流程：**

1. `Shift + Tab` 切换到 **Plan** 模式
2. 描述功能需求（尽量具体）
3. 回答 Agent 的澄清问题
4. 审查生成的计划（虚拟文件，可直接编辑）
5. 满意后点击 **Build** 开始编码
6. 计划可 **Save to workspace** 供团队参考

### 4.5 Debug 模式

专为难以定位的 Bug 设计，Agent 会先提出假设、添加诊断日志，用运行时信息定位根因，而非盲目改代码。

**示例提示词：**

```
用户报告 /api/orders 偶尔返回 500 错误。
错误日志如下：[粘贴日志]
请分析根因，添加诊断日志，修复后跑相关测试。
```

### 4.6 模式选型速查

| 你想… | 用… |
|--------|-----|
| 只问不改 | Ask（`Cmd+L` + `Shift+Tab`） |
| 改当前选中代码 | Inline Edit（`Cmd+K`） |
| 跨文件 / 跑命令 | Agent（`Cmd+I`） |
| 大功能先审方案 | Plan（`Shift+Tab`） |
| 难查的 Bug | Debug（`Shift+Tab`） |

---

## 5. 上下文与 @ 提及

### 5.1 代码库索引

Cursor 会对工作区进行 **语义索引**，使 Agent 能搜索和理解大型代码库。

**忽略文件配置：**

- `.cursorignore`：排除不参与索引的文件（类似 `.gitignore`）
- `.cursorindexingignore`：更细粒度的索引排除控制

### 5.2 @ 提及类型

在聊天输入框输入 `@` 可附加上下文：

| 提及 | 说明 |
|------|------|
| `@文件` / `@文件夹` | 精确附加文件或目录（文件夹后可输入 `/` 深入） |
| `@Codebase` | 语义搜索整个代码库（`Cmd/Ctrl + Enter` 也可触发） |
| `@Docs` | 搜索已索引文档（可通过 `@Docs > Add new doc` 添加） |
| `@Web` | 联网搜索最新信息 |
| `@Git` | Git 相关信息 |
| `@Terminals` | 附加终端输出 |
| `@Past Chats` | 引用历史对话上下文 |
| `@Commit (Diff of Working State)` | 未提交的变更 |
| `@Branch (Diff with Main)` | 与主分支的完整 diff |
| `@Browser` | 内置浏览器上下文 |

### 5.3 使用建议

- **已知相关文件**：主动 `@` 精确附加，减少 Agent 搜索时间
- **不确定位置**：让 Agent 自行搜索代码库即可
- **大型 diff 审查**：使用 `@Branch (Diff with Main)` 附加完整变更

---

## 6. Rules、Skills 与 Commands

### 6.1 Rules（规则）

规则是给 Agent 的 **持久化指令**，避免每次重复说明项目规范。

#### 规则类型与优先级

| 类型 | 位置 | 作用域 | 优先级 |
|------|------|--------|--------|
| **Team Rules** | Cursor 团队仪表盘 | 全组织 | 最高 |
| **Project Rules** | `.cursor/rules/` | 当前项目（可 Git 共享） | 中 |
| **User Rules** | Cursor Settings → Rules | 本机所有项目 | 较低 |
| **AGENTS.md** | 项目根目录 | 自动加载 | 与项目规则配合 |
| **.cursorrules** | 项目根目录 | ⚠️ 已弃用，建议迁移 | — |

#### 创建项目规则

1. `Cmd/Ctrl + Shift + P` → 输入 `New Cursor Rule`
2. 命名规则，如 `api-conventions`
3. 用 Markdown 编写指令
4. 选择应用方式：

| 类型 | 说明 |
|------|------|
| Always Apply | 每次对话都包含 |
| Apply Intelligently | Agent 判断是否相关 |
| Apply to Specific Files | 匹配 glob（如 `*.tsx`） |
| Apply Manually | 仅 `@规则名` 时生效 |

#### AGENTS.md 示例

在项目根目录创建 `AGENTS.md`：

```markdown
# 项目说明

- 新文件使用 TypeScript，严格模式
- API 遵循 RESTful 规范，错误格式见 src/types/error.ts
- React 组件不超过 200 行，复杂逻辑提取为 Hook
- 提交前必须跑 `npm run lint && npm test`
- 不要修改 src/legacy/ 目录下的文件
```

### 6.2 Skills（技能）

Skills 是比 Rules 更详细的 **多步骤工作流**，按需调用。

- **位置**：`.cursor/skills/技能名/SKILL.md`
- **调用**：`/技能名` 或 `@技能名`
- **创建**：在聊天输入 `/create-skill`
- **迁移**：输入 `/migrate-to-skills`（Cursor 2.4+）

| 对比 | Rules | Skills |
|------|-------|--------|
| 长度 | 短（< 500 行） | 通常更长、分步骤 |
| 触发 | 自动或按文件匹配 | 手动 `/` 调用 |
| 示例 | "新文件用 TypeScript" | "部署到 staging 的完整流程" |

### 6.3 Commands（斜杠命令）

可复用的 `/` 斜杠命令，快速触发常用工作流：

| 命令 | 说明 |
|------|------|
| `/multitask` | 并行启动多个子代理 |
| `/in-cloud` | 在云端 VM 中执行任务 |
| `/babysit` | 云端长跑任务监控 |
| `/review` | 本地代码审查 |
| `/review-bugbot` | 使用 Bugbot 规则审查 |
| `/debug` | 进入 Debug 模式 |
| `/summarize` | 总结当前对话 |
| `/automate` | 创建自动化工作流 |
| `/worktree` | 创建隔离的 git worktree |
| `/best-of-n` | 多模型并行对比结果 |

---

## 7. MCP 外部工具集成

MCP（Model Context Protocol）让 Cursor Agent 连接 **外部工具与数据源**，如数据库、GitHub、Notion、Figma、Jira 等。

### 7.1 配置位置

| 范围 | 路径 |
|------|------|
| 全局 | `~/.cursor/mcp.json` |
| 项目 | `.cursor/mcp.json`（可提交 Git，团队共享） |

### 7.2 配置示例

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${env:GITHUB_TOKEN}"
      }
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${env:DATABASE_URL}"
      }
    },
    "remote-api": {
      "url": "https://api.example.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:API_TOKEN}"
      }
    }
  }
}
```

### 7.3 配置步骤

1. `Cmd/Ctrl + Shift + J` → **MCP & Integrations** → Add new MCP server
2. 或直接编辑 `.cursor/mcp.json`
3. 设置所需环境变量（如 `GITHUB_TOKEN`）
4. 重启 Cursor，在 Customize 侧边栏确认 MCP 已启用
5. 在 Agent 中测试：「列出我 GitHub 上最近的 open PR」

### 7.4 常用 MCP 服务器

| 服务器 | 用途 |
|--------|------|
| `@modelcontextprotocol/server-github` | GitHub PR、Issue、代码操作 |
| `@modelcontextprotocol/server-postgres` | 数据库查询与 Schema 读取 |
| `@modelcontextprotocol/server-slack` | Slack 消息搜索 |
| `@modelcontextprotocol/server-notion` | Notion 文档读取 |
| `@modelcontextprotocol/server-figma` | Figma 设计稿读取 |

### 7.5 调试 MCP

- 打开输出面板 → 选择 `MCP Logs` 查看连接日志
- 先在终端手动运行 MCP 命令，确认服务能正常启动
- 检查环境变量是否正确设置

---

## 8. Agents Window 与 Cloud Agents

### 8.1 Agents Window

**Agents Window**（Cursor 3+）是 Agent 优先的工作区，通过 `Cmd+Shift+P` → `Open Agents Window` 打开。

**独有功能：**

- **多工作区** 统一管理 Agent
- **并行 Cloud Agents**：云端 VM 同时跑多个任务
- **本地 ↔ 云端** 一键切换
- **Cloud Subagents**：`/in-cloud`、`/babysit` 在云端 VM 长跑
- **Worktrees**：Git 隔离检出，每个任务独立文件树
- 新版 diff 视图、PR 管理

### 8.2 多 Agent 并行

```
/multitask          → 并行子代理，不排队
/worktree           → 创建隔离 git worktree
/best-of-n          → 多模型并行对比结果
Plan → Build in Parallel → 独立步骤并行执行
```

子代理内置类型：research（调研）、shell（终端）、browser（浏览器）；自定义子代理放在 `.cursor/agents/`。

### 8.3 Cloud Agents

在 [cursor.com/agents](https://cursor.com/agents) 或 Agents Window 中启动，运行在 **隔离 Ubuntu VM** 上。

**能力：**

- 完整桌面环境 + **Computer Use**（键鼠操作浏览器）
- 启动 dev server、点击 UI 流、截图/录屏验证
- 自动修复 PR 的 CI 失败（GitHub Actions）
- 支持 MCP（HTTP 推荐）
- 多仓库环境、Docker/Tailscale/Cloudflare Tunnel
- 产出 Artifact（截图、视频）附在 PR 上

**环境配置：**

```json
// .cursor/environment.json
{
  "install": "npm install",
  "start": "npm run dev",
  "test": "npm test"
}
```

Secrets 在 [cursor.com/dashboard](https://cursor.com/dashboard) 管理，不要在代码中硬编码。

**使用流程：**

1. 访问 cursor.com/agents 或打开 Agents Window
2. 连接 GitHub/GitLab 仓库
3. 配置 Cloud Environment（依赖、Secrets、启动命令）
4. 描述任务，Agent 在云端 VM 执行
5. 审查 PR、Artifact（截图/视频），必要时远程桌面接管

---

## 9. 终端集成与安全策略

### 9.1 Agent 自动执行命令

路径：**Settings → Agents → Approvals & Execution**

| 模式（3.6+） | 行为 | 适用 |
|--------------|------|------|
| **Auto-review**（推荐默认） | 白名单直接执行；其余尽量沙箱运行；高风险由分类器审查 | 大多数用户 |
| **Allowlist** | 仅白名单命令免批准 | 需要确定性控制 |
| **Run Everything** | 全部自动执行，无提示 | 完全信任环境（不推荐） |

### 9.2 沙箱（Sandbox）

- macOS：Seatbelt；Linux：Landlock + seccomp（需 Kernel 6.2+）
- 配置：`~/.cursor/sandbox.json` 或 `项目/.cursor/sandbox.json`

```json
{
  "network": {
    "allowedDomains": ["api.github.com", "registry.npmjs.org"]
  }
}
```

### 9.3 终端内 AI

聚焦终端 → `Cmd/Ctrl + K` → 用自然语言描述命令 → 执行

**Cursor CLI**（在终端独立使用 Agent）：

```bash
curl https://cursor.com/install -fsS | bash
cursor agent "修复所有 TypeScript 类型错误"
```

---

## 10. Bugbot 代码审查

Bugbot 自动审查 Pull Request，发现 Bug、安全问题和代码质量问题。

### 10.1 触发方式

| 方式 | 说明 |
|------|------|
| 自动 | PR 每次更新自动审查 |
| 手动 | PR 评论 `cursor review` 或 `bugbot run` |
| 本地 | `/review-bugbot` 或 `/review`（Cursor 3.7+） |

### 10.2 配置审查规则

在项目根目录创建 `.cursor/BUGBOT.md`：

```markdown
# Bugbot 审查规则

- 检查所有 API 端点是否有输入验证
- 不允许硬编码密钥或 token
- 数据库查询必须使用参数化查询
- 新增函数必须有对应的单元测试
```

**规则优先级**：Team Rules → 仓库规则 → `.cursor/BUGBOT.md` → User Rules

### 10.3 高级功能

- **Autofix**：自动启动 Cloud Agent 修复发现的问题
- **Effort Levels**：Default / High / Custom（按用量计费）
- **Incremental Review**：只审查自上次审查后的变更
- 学习规则：PR 评论 `@cursor remember [事实]`

---

## 11. Customize 与 Plugins（3.9）

Cursor 3.9 引入 **Customize** 侧边栏，统一管理所有 AI 配置。

| 组件 | 说明 |
|------|------|
| **Plugins** | 可分发 bundle（含 rules、skills、MCP、hooks 等） |
| **Rules / Skills / MCP / Subagents / Hooks / Commands** | 用户 / 团队 / 工作区三级作用域 |
| **Marketplace Leaderboard** | 团队热门扩展排行 |
| **Plugin Canvases** | 共享设置模板（如 Hex、Atlassian） |
| **Team Marketplaces** | 从 GitHub、GitLab、BitBucket、Azure DevOps 导入 |

访问 [cursor.com/marketplace](https://cursor.com/marketplace) 浏览社区插件。

---

## 12. 典型使用案例

### 案例 1：编写新功能

**场景**：为电商站添加「购物车优惠券」功能

```
Plan 模式操作：
1. 「为购物车添加优惠券功能：输入码验证、折扣计算、与结账流程集成。
   先出实现计划，不要写代码。」
2. 审查计划，确认 API 设计、前端组件、测试覆盖
3. 点击 Build，Agent 自动：读相关模块 → 改 API/前端/测试 → 跑 test
4. 推送前运行 /review-bugbot 本地审查
```

**提示词技巧**：说明验收标准、不要动的模块、测试要求。

---

### 案例 2：调试难复现 Bug

**场景**：生产环境偶发 500 错误

```
Debug 模式：
「用户报告 /api/orders 偶尔返回 500。错误日志如下：
[粘贴日志内容]
请分析根因，添加诊断日志，修复后跑相关测试。」
```

Debug 模式会先提出假设、加日志、用运行时信息定位，而非盲目改代码。

---

### 案例 3：大规模重构

**场景**：将 class 组件迁移到 Hooks

```
Agent 模式 + @src/legacy/：
「将 src/legacy/ 下所有 class 组件重构为函数组件 + Hooks。
保持行为不变，每改一个文件跑对应测试。
完成后生成迁移报告。」
```

大重构可用 `/multitask` 或 Cloud Agent 并行处理多个文件。

---

### 案例 4：代码审查

| 层级 | 方式 |
|------|------|
| 本地推送前 | `/review-bugbot` 或 `/review` |
| PR 自动 | Bugbot 自动审查 |
| 人工辅助 | Ask 模式：「审查 @Branch (Diff with Main) 的安全问题」 |

---

### 案例 5：生成文档

```
Agent 模式：
「为 src/api/ 下所有公开函数生成 JSDoc，
并更新 README 的 API 章节，包含请求/响应示例。」

Ask 模式：
「根据 @src/ 总结系统架构，输出 Mermaid 架构图。」
```

---

### 案例 6：补充测试

```
Agent 模式：
「为 UserService 补充单元测试，覆盖以下边界情况：
- 空邮箱注册
- 重复注册
- token 过期刷新
使用项目现有 Jest 配置，mock 数据库连接。」
```

---

### 案例 7：前端 UI 验证

```
Agent 模式：
「启动 dev server，在浏览器打开 /checkout 页面，
测试优惠券输入框的校验逻辑（空值、无效码、过期码），
截图有问题的地方并修复。」
```

---

### 案例 8：跨仓库协作（Cloud Agent）

```
Cloud Agent + 多仓库环境：
「在 frontend 仓库添加新 API 调用，
在 backend 仓库实现对应 endpoint，
两边测试通过后各开 PR，附上集成测试截图。」
```

---

### 案例 9：CI 失败自动修复

```
Cloud Agent（PR 触发）：
「GitHub Actions CI 失败了，日志如下：[粘贴]
请分析失败原因，修复代码，推送后确认 CI 通过。」
```

---

### 案例 10：数据库 Schema 迁移

```
Agent + MCP（Postgres MCP）：
「读取当前数据库 Schema，生成将 users 表添加 phone 字段的迁移脚本，
包含回滚脚本，并更新对应的 TypeScript 类型定义。」
```

---

## 13. 快捷键速查表

> Windows/Linux 将 `Cmd` 换为 `Ctrl`，`Opt` 换为 `Alt`

### AI 专用快捷键

| 操作 | Mac | Windows/Linux |
|------|-----|---------------|
| 打开 Agent 侧栏 | `Cmd + I` | `Ctrl + I` |
| 打开 Chat 侧栏 | `Cmd + L` | `Ctrl + L` |
| 行内编辑 | `Cmd + K` | `Ctrl + K` |
| 模式菜单 | `Cmd + .` | `Ctrl + .` |
| 切换 Agent 模式 | `Shift + Tab` | `Shift + Tab` |
| 切换 AI 模型 | `Cmd + /` | `Ctrl + /` |
| 接受 Tab 建议 | `Tab` | `Tab` |
| 逐词接受 Tab | `Cmd + →` | `Ctrl + →` |
| 提交 Agent 提示 | `Return` | `Return` |
| 接受 Inline/Agent 修改 | `Cmd + Enter` | `Ctrl + Enter` |
| 拒绝所有修改 | `Cmd + Backspace` | `Ctrl + Backspace` |
| 取消 AI 生成 | `Cmd + Shift + Backspace` | `Ctrl + Shift + Backspace` |
| 选中代码加入 Chat | `Cmd + Shift + L` | `Ctrl + Shift + L` |
| 代码库语义搜索 | `Cmd + Enter`（Chat 中） | `Ctrl + Enter` |
| Cursor 设置 | `Cmd + Shift + J` | `Ctrl + Shift + J` |
| 打开 Agents Window | `Cmd + Shift + P` → 搜索 | 同上 |

### VS Code 继承快捷键

Cursor 继承 VS Code 全部快捷键，常用包括：

| 操作 | Mac | Windows/Linux |
|------|-----|---------------|
| 命令面板 | `Cmd + Shift + P` | `Ctrl + Shift + P` |
| 快速打开文件 | `Cmd + P` | `Ctrl + P` |
| 全局搜索 | `Cmd + Shift + F` | `Ctrl + Shift + F` |
| 切换侧边栏 | `Cmd + B` | `Ctrl + B` |
| 集成终端 | `` Cmd + ` `` | `` Ctrl + ` `` |
| 查看全部快捷键 | `Cmd + R` 然后 `Cmd + S` | `Ctrl + R` 然后 `Ctrl + S` |

---

## 14. 最佳实践

### 14.1 提示词技巧

- **具体明确**：说明文件路径、期望行为、验收标准，避免「优化一下」这类模糊指令
- **分步执行**：大任务用 Plan 模式，或拆成多个 Agent 会话
- **新任务新对话**：切换模式或任务时开新 Chat，避免上下文污染
- **善用 @**：已知相关文件时精确附加；不确定时让 Agent 自行搜索

**好的提示词示例：**

```
❌ 差：「优化这个函数」
✅ 好：「将 src/utils/auth.ts 中的 validateToken 函数改为 async/await，
      添加 token 过期检查，错误时抛出 AuthError，
      保持现有单元测试全部通过。」
```

### 14.2 项目配置

- 用 `.cursor/rules/` + `AGENTS.md` 固化团队规范，提交到 Git
- `.cursorignore` 排除 `node_modules`、密钥文件、生成物
- 将 `.cursor/mcp.json`、`sandbox.json` 提交 Git（Secrets 用环境变量 `${env:VAR}`）
- Cloud Agent 在 `AGENTS.md` 中写清环境启动与测试命令

### 14.3 安全建议

- Run Mode 使用 **Auto-review**，勿轻易选 Run Everything
- MCP 只安装可信来源；API Key 用 `${env:VAR}` 而非硬编码
- 企业环境启用 Privacy Mode、MCP 白名单、团队规则
- 始终审查 Agent 的 diff，用 Restore Checkpoint 快速回滚错误修改

### 14.4 效率提升

| 场景 | 推荐方式 |
|------|----------|
| 日常编码 | Tab 补全 + Inline Edit（`Cmd+K`） |
| 单文件小改动 | Inline Edit |
| 跨文件功能 | Agent 模式 |
| 大型功能 | Plan 模式 → Build |
| 并行多任务 | `/multitask` 或 Cloud Agent |
| 关电脑也能跑 | Cloud Agent + `/babysit` |
| 推送前审查 | `/review-bugbot` |
| 省额度 | Auto 或 Composer 2.5 路由 |

### 14.5 模型选择建议

| 场景 | 建议 |
|------|------|
| 日常编码 | Auto 或 Composer 2.5（省额度） |
| 复杂架构 / 难 Bug | Premium 或手动选 Opus / GPT-5.5 等 |
| 超大代码库 | Max Mode（注意额度消耗） |
| 团队统一 | 仪表盘设默认模型 |

---

## 15. 定价方案

### 15.1 个人计划

| 计划 | 月费 | 含 API 用量 | Auto + Composer 2.5 |
|------|------|-------------|---------------------|
| **Hobby（免费）** | $0 | 有限 | 有限 |
| **Pro** | $20/月 | $20 | 大量包含用量 |
| **Pro+** | $60/月 | $70 | 大量包含用量 |
| **Ultra** | $200/月 | $400 | 大量包含用量 |

**所有付费个人计划包含**：无限 Tab 补全、全模型 Agent 用量、Bugbot 访问、Cloud Agents 访问。

### 15.2 用量池说明

两套独立用量池，每月随账单周期重置：

1. **Auto + Composer 池**：选 Auto 或 Composer 2.5 时使用，成本更低
2. **API 池**：手动选择特定模型时，按该模型 API 价格计费

| 路由 | 费率 |
|------|------|
| **Auto** | 输入 $1.25/1M、输出 $6.00/1M tokens |
| **Composer 2.5** | 输入 $0.5/1M、输出 $2.5/1M tokens |
| **Premium** | 自动选最强模型，按该模型 API 费率 |

### 15.3 团队与企业

| 计划 | 月费 | 说明 |
|------|------|------|
| **Teams Standard** | $40/用户/月 | 集中计费、共享规则、SSO、Bugbot |
| **Teams Premium** | $120/用户/月 | Standard 的 5 倍 Agent 额度 |
| **Enterprise** | 定制 | 池化用量、SCIM、审计日志、合规、私有连接 |

---

## 16. 常见问题

| 问题 | 处理方法 |
|------|----------|
| Tab 补全不工作 | 检查右下角 Tab 指示器；Settings → Tab |
| Agent 乱改不相关文件 | 加强 Rules；用 Plan 先审；检查 `.cursorignore` |
| MCP 连接失败 | Output → MCP Logs；重启 Cursor；检查 `mcp.json` 语法 |
| 终端输出乱码 | 复杂 Shell 主题在 `CURSOR_AGENT` 环境时禁用 |
| Rules 不生效 | 确认规则类型；Intelligent 类型需写 description |
| 额度消耗过快 | 切换到 Auto 或 Composer 2.5；避免不必要的 Max Mode |
| Cloud Agent 环境启动失败 | 检查 `.cursor/environment.json`；在仪表盘查看日志 |
| 想回滚 Agent 修改 | 使用 Restore Checkpoint 回滚到某条消息之前 |

---

## 17. 官方资源

| 资源 | 链接 |
|------|------|
| 官方文档 | [cursor.com/docs](https://cursor.com/docs) |
| 中文文档 | [cursor.com/cn/docs](https://cursor.com/cn/docs) |
| 定价 | [cursor.com/pricing](https://cursor.com/pricing) |
| 更新日志 | [cursor.com/changelog](https://cursor.com/changelog) |
| Cloud Agents | [cursor.com/agents](https://cursor.com/agents) |
| CLI | [cursor.com/cli](https://cursor.com/cli) |
| Marketplace | [cursor.com/marketplace](https://cursor.com/marketplace) |
| MCP 社区目录 | [cursor.directory](https://cursor.directory) |
| 仪表盘 | [cursor.com/dashboard](https://cursor.com/dashboard) |

---

*本教程基于 Cursor 官方文档整理，功能随版本更新可能有所变化。建议关注 [Changelog](https://cursor.com/changelog) 获取最新信息。*
