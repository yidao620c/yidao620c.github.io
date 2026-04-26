---
title: Claude Code快速入门笔记
date: 2026-04-21 22:11:35 +0800
toc: true
categories: [ AI ]
tags: [ Claude Code ]
---
Claude Code‌是 Anthropic 公司推出的终端 AI 编程助手，可通过命令行界面直接读取、编辑代码库并执行开发任务。当今世界AI编程助手稳坐第一。

- 智能文件操作‌：自动读取和分析项目文件，理解代码结构和上下文，无需手动添加文件到对话中 。‌ 
- 全项目上下文‌：理解整个代码库的架构和依赖关系，支持跨文件追踪代码依赖 。‌ 
- 自主任务执行‌：可读取、编辑、创建文件，执行 shell 命令，运行测试和构建流程 。‌

<!--more-->

## 安装脚本

打开Windows PowerShell
```
irm https://claude.ai/install.ps1 | iex
```

## 配置文件

下载cc-switch，添加智普GLM供应商，配置好API Key，选择模型名称统一为`glm-5.1`。
按智普官网文档，还要新建一个`.claude.json`，内容
```json
{
  "hasCompletedOnboarding": true
}
```

## 安装Karpathy准则
一个单一的 `CLAUDE.md` 文件，用于改善 Claude Code 的行为，源自 Andrej Karpathy 的观察 关于 LLM 编码陷阱的总结。
项目地址：https://github.com/forrestchang/andrej-karpathy-skills

**选项 A：Claude Code 插件（推荐）**

在 Claude Code 中，首先添加插件市场：
```
/plugin marketplace add https://github.com/forrestchang/andrej-karpathy-skills.git
```

然后安装插件：
```
/plugin install andrej-karpathy-skills@karpathy-skills
```

这会将指南安装为 Claude Code 插件，使其在你所有项目中可用。

**选项 B：CLAUDE.md（按项目）**

新项目：
```bash
curl -o CLAUDE.md https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md
```

已有项目（追加）：
```bash
echo "" >> CLAUDE.md
curl https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md >> CLAUDE.md
```

**如何判断它在起作用**

如果你看到以下情况，说明这些指南正在发挥作用：

- **diff 中不必要的改动更少** —— 只有请求的改动出现
- **因过度复杂而导致的重写更少** —— 代码第一次就写得简洁
- **澄清问题在实现之前提出** —— 而不是在犯错之后
- **干净、精简的 PR** —— 没有顺带的重构或"改进"

这些指南设计用于与项目特定指令合并。将它们添加到你现有的 `CLAUDE.md` 或创建一个新的。
对于项目特定规则，添加如下章节：
```markdown
## 项目特定指南

- 使用 TypeScript 严格模式
- 所有 API 端点必须有测试
- 遵循 `src/utils/errors.ts` 中现有的错误处理模式
```

## 安装技能
两个SKILL市场网站，一个中文一个英文：
- https://www.agentskills.in/zh-CN
- https://skills.sh/

安装技能前提是必须得能访问GitHub网站。

配置Git代理：
```
git config --global http.proxy http://127.0.0.1:8086
git config --global https.proxy http://127.0.0.1:8086
git config --global http.sslVerify false

#取消
git config --global --unset http.proxy
git config --global --unset https.proxy
```

`skills.sh`安装方式：
```
npx skills list     #项目级
npx skills list -g  #全局级
npx skills add https://github.com/anthropics/skills --skill pdf
npx skills add https://github.com/anthropics/skills --skill pdf -g
```

`agentskills.in`安装方式：
```
npx agent-skills-cli install @anthropics/pdf -g
```

安装技能列表：
- **find-skills**：当用户询问"如何做某事"、"寻找某技能"或希望扩展功能时，帮助发现并安装智能体技能。
- **skill-creator**：创建有效技能指南。当用户希望创建新技能或更新现有技能时使用。
- **frontend-design**：创建具有高品质设计、独特且适用于生产环境的前端用户界面。
- **web-design-guidelines**：审查UI代码以确保符合Web界面指南。当被要求“审查我的UI”、“检查可访问性”、“审计设计”时使用。
- **ui-ux-pro-max**：提供 UI/UX 设计智能与实现指导，帮助打造精美界面。适用于前端 UI生成/评审/改进。
- **agent-browser**：无头浏览器自动化CLI，允许AI代理通过结构化命令执行页面导航、点击、输入和快照操作。
- **playwright-cli**：自动化浏览器交互，用于网页测试、表单填写、截图和数据提取。
- **pptx**：使用设计指导和质量保证工作流程创建、编辑、阅读和操作 PowerPoint 演示文稿。
- **xlsx**：创建、编辑和分析带有公式、格式和计算正确的Excel电子表格。
- **pdf**：全面的PDF处理功能，包括文本提取、合并、拆分、表单填写及OCR识别能力。
- **self-improving-agent**：记录经验教训、错误及修正以实现持续改进。
- **planning-with-filesx**：使用文件做任务规划，用于组织和跟踪复杂任务的进度。创建`task_plan.md`、`findings.md`和`progress.md`。

## 安装Superpowers

打开Claude Code后执行
```
# 注册市场
/plugin marketplace add obra/superpowers-marketplace
# 安装插件
/plugin install superpowers@superpowers-marketplace
```

## 安装OpenSpec
在终端中全局安装 OpenSpec：
```
npm install -g @fission-ai/openspec@latest
```

安装后使用 `openspec --version` 检查版本号，确认安装成功。

设置环境变量关闭遥测：
```
OPENSPEC_TELEMETRY=0
```

**项目初始化**

- 进入项目：`cd your-project`
- 执行初始化：`openspec init`
- 选择工具：在交互界面中，通过方向键和空格键选择 Claude Code。
- 初始化完成后，会在项目根目录创建 `openspec/` 目录和针对 Claude Code 的斜杠命令（Slash Commands）。

**核心工作流**

OpenSpec 将开发工作拆解为下面4个阶段：

* 📝 提案：描述目标，AI 生成任务清单，人类审查确认。在 Claude Code 中输入 `/opsx:propose` 后跟功能描述。
* ⚙️ 实施：命令 AI 自动完成任务清单。确认提案后，使用 `/opsx:apply` 进入编码阶段。
* ✅ 验证（可选）：利用 `openspec-validate-change` 技能检查代码与规范的匹配度。
* 🗄️ 归档：功能完成后归档变更，更新主规范，让 `openspec/specs/` 保持最新。使用 `/opsx:archive` 命令完成变更归档。

