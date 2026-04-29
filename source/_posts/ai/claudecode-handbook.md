---
title: Claude Code使用指南
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

后续如果更新到最新版本：
```
claude update
```

## 配置文件

下载cc-switch，添加智普GLM供应商，配置好API Key，选择模型名称统一为`glm-5.1`。
按智普官网文档，还要新建一个`~/.claude/.claude.json`，内容
```json
{
  "hasCompletedOnboarding": true
}
```

主配置文件为`~/.claude/settings.json`，可配置编辑文件时不需要确认。
```json
{
  "permissions": {
    "defaultMode": "acceptEdits"
  }
}
```

完全不需要任何确认，则使用启动参数：
```
claude --dangerously-skip-permissions
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
- **planning-with-files**：使用文件做任务规划，用于组织和跟踪复杂任务的进度。创建`task_plan.md`、`findings.md`和`progress.md`。
- **tavily-search**：面向AI和RAG（检索增强生成）的搜索。返回结构化/摘要信息，附带来源。

安装命令列表：
```
npx skills add https://github.com/vercel-labs/skills --skill find-skills -g
npx skills add https://github.com/anthropics/skills --skill skill-creator -g
npx skills add https://github.com/anthropics/skills --skill frontend-design -g
npx skills add https://github.com/vercel-labs/agent-skills --skill web-design-guidelines -g
npx skills add https://github.com/nextlevelbuilder/ui-ux-pro-max-skill --skill ui-ux-pro-max -g
npx skills add https://github.com/vercel-labs/agent-browser --skill agent-browser -g
npx skills add https://github.com/microsoft/playwright-cli --skill playwright-cli -g
npx skills add https://github.com/anthropics/skills --skill pptx -g
npx skills add https://github.com/anthropics/skills --skill xlsx -g
npx skills add https://github.com/anthropics/skills --skill pdf -g
npx skills add https://github.com/charon-fan/agent-playbook --skill self-improving-agent -g
npx skills add https://github.com/othmanadi/planning-with-files --skill planning-with-files -g
npx skills add https://github.com/tavily-ai/skills --skill tavily-search -g
```

## Superpowers

Superpowers 是一个强化编程工作流的插件（Skills 框架），核心是给 Claude 套上软件工程纪律，让它按规范流程写代码，而不是即兴输出。
固定的7 步标准化开发流程，全程按顺序走，每个阶段对应一个独立的技能（Skill），按顺序自动触发：
- brainstorming：苏格拉底式提问，澄清需求，输出设计文档
- using-git-worktrees：创建隔离的 Git 工作区，验证测试基线
- writing-plans：把工作拆成 2-5 分钟的小任务，每个任务有完整代码和验证步骤。
- subagent-driven-development：每个任务派一个独立子代理执行，两阶段审查。
- test-driven-development：强制 RED-GREEN-REFACTOR：先写失败测试 → 实现 → 通过 → 重构。
- requesting-code-review：审查代码质量和规格合规性，Critical 问题阻断进度
- finishing-a-development-branch：验证测试全部通过，合并 / PR / 清理

打开Claude Code后执行
```
# 注册市场
/plugin marketplace add obra/superpowers-marketplace
# 安装插件
/plugin install superpowers@superpowers-marketplace
```

## OpenSpec
OpenSpec 是 Fission AI 团队做的规约框架。核心理念就一句话：写代码之前，先把要做什么写清楚，而且要写成 AI 可读的格式。
它不是让你写 PRD 文档那种纯文本。OpenSpec 的产物是结构化的 Markdown 文件，AI 编程助手可以直接读取、理解、执行。

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

## 优秀实践

### 安装Karpathy准则
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

### DESIGN.md
`VoltAgent/awesome-design-md`是一个前端页面设计`DESIGN.md`风格库，给AI补一套审美说明书。
地址：https://github.com/VoltAgent/awesome-design-md

使用非常简单，先打开网站：https://getdesign.md/ 找到一个你想要的UI风格主题，把对应文件夹中的`DESIGN.md`复制到项目根目录。
然后你就可以跟AI Agent说：
```
build me a page that looks like this
```

### 【SKILL】tavily-search
首先免费获取 API Key：访问 [Tavily官网](https://tavily.com/) 注册，可以免费获取一个月度1000次查询的API Key。
然后设置环境变量TAVILY_API_KEY为刚刚申请到的API Key。完成设置后，请完全退出并重启 Claude Code，让配置生效。

AI 自动模式（推荐）：直接在对话中提出你的需求，AI 会根据情况自动调用搜索。这是最自然直观的方式。
示例提问：“帮我搜索一下过去一周内，关于AI Agent领域的重要融资事件”

命令手动调用：你也可以用明确的斜杠命令，指定让它执行哪种任务。例如：
- 搜索：/tavily-search 2026年最新的前端框架趋势
- 提取网页：/tavily-extract https://example.com/blog/post
- 爬取站点：/tavily-crawl https://docs.example.com
- 深度研究：/tavily-research 大语言模型的市场竞争格局

### 【SKILL】洁癖
大家现在也都知道，在很多时候，你的Agent之所以越用越笨，其实是因为，你的上下文过于混乱。
上下文不止是你跟你的Agent在单次对话中的聊天记录，也包括你这个项目里面的，各种文档，还有约束，还有记忆。

它的功能是在开发阶段结束后，自动审查并同步项目文档（`CLAUDE.md`、`README.md`、`docs/`）和 Agent 记忆，确保知识体系与代码保持一致。

如果你用Agent开发过任何东西，你就知道，一个项目里的知识其实分三层，每一层服务的人不太一样。

- 第一层是Agent自己的记忆系统，过去的聊天记录、项目的隐性知识。
- 第二层是项目根目录的`CLAUDE.md`，给AI自己看的，项目约定、结构、红线、路由清单等等。
- 第三层是`docs/`目录和`README.md`，给其他人看的，比如Agent、同事、下游开发者等等，比如接入指南、架构说明、运维手册等等。

项目地址：https://github.com/KKKKhazix/khazix-skills

打开Claude Code后直接跟它讲
```
帮我安装这个 skill：https://github.com/KKKKhazix/khazix-skills/tree/main/neat-freak
```
触发提示词，退出会话的时候最好都执行一遍：
```
/neat            # 直接命令
整理一下          # 自然语言
同步一下          # 自然语言
sync up          # English
```

### 【SKILL】agent-browser

安装 CLI 工具
```
npm install -g agent-browser
agent-browser install  # 自动下载 Chromium
```

直接用中文指令，Claude 自动转命令：
- 测试表单：" 打开 http://localhost:3000/register，填 test@x.com 和 123456，提交并截图错误提示 "
- UI 验证："访问首页，全屏截图，检查导航栏是否正常显示"
- 流程自动化："登录后台，点击‘用户列表’，滚动到底部，截图所有数据"

### 【SKILL】playwright-cli
安装 CLI 工具
```
# 1. 安装 Playwright
npm install -g playwright
# 2. 安装浏览器（Chromium）
playwright install chromium
```

你可以直接在 Claude 里发指令，让它用 Playwright CLI 帮你写自动化脚本：
```
用 playwright-cli 打开 localhost:3000，点击登录按钮，输入账号密码，截图验证。
```


