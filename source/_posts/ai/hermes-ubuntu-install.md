---
title: Ubuntu24安装Hermes Agent
date: 2026-04-19 07:47:29 +0800
toc: true
categories: [ AI编程 ]
tags: [ Hermes ]
---
Hermes Agent 是 Nous Research 今年 2 月底开源的 AI 智能体框架。

上线不到两个月，GitHub 星标冲到了三万多。社区里不少人把它称为 OpenClaw 上线以来，第一个真正意义上的竞争对手。

OpenClaw 的核心是一个 Gateway。网关守护进程，负责统一管理会话、路由消息、连接各种聊天平台。
你可以理解成一个调度中心，把所有聊天应用接到 AI Agent 上。龙虾解决的核心问题是：怎么把消息送到 Agent

Hermes 不太关心这个，它更在意的是：Agent 怎么变得越来越强。官方管这叫 `closed learning loop`，闭环学习循环。
整个框架围绕的就是一件事——让 Agent 在使用过程中自我进化。

当它完成一个复杂任务——通常涉及五次以上工具调用——它不是做完就算了。
它会把整个过程沉淀成一份结构化的技能文档，存成 Markdown 文件，放在 `~/.hermes/skills/` 目录下。
下次遇到类似任务？直接加载这份技能文档，不用从头解决。
更狠的是，这些技能在使用过程中会自我迭代。Agent 执行某个技能时发现了更好的方法，它会自动更新技能文档。不需要你手动维护。

<!--more-->

## 安装脚本

一行命令搞定安装。安装脚本会自动处理 Python、Node.js 和所有依赖。MacOS、Linux、WSL2 都能跑。 
```
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

装完之后，按照安装指引配置后台模型，这里我选择的是最新的MiniMax2.7。

## 配置飞书接入
Hermes的飞书接入简单到爆炸，它在配置指导里面会让你扫个二维码就搞定了，你就有了一个能聊天的Hermes机器人。
你不需要去飞书平台去创建应用、配置权限、设置事件与回调。这些它都会自动给你做完。

将Hermes Gateway安装为系统服务开机自启动
```
sudo /home/ubuntu/.local/bin/hermes gateway install --system
sudo systemctl restart hermes-gateway
# 查看日志文件
tail -200f ~/.hermes/logs/agent.log
```

## 迁移OpenClaw配置

一行命令搞定
```
hermes claw migrate --dry-run
hermes claw migrate --overwrite
```

底层模型不会覆盖，如果要修改模型需要执行
```
hermes model
```
还需要编辑文件`~/.hermes/.env`把密钥写进去。

