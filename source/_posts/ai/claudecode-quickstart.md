---
title: Claude Code快速入门笔记
date: 2026-04-21 22:11:35 +0800
toc: true
categories: [ AI编程 ]
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

