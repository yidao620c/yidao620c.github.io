---
title: OpenClaw使用指南
date: 2026-04-07 17:15:22 +0800
toc: true
categories: [ AI编程 ]
tags: [ OpenClaw ]
---

记录一些自己在使用OpenClaw过程中的笔记。

## 定时任务（Cron）

想让龙虾每天早上给你推新闻？那就需要定时任务。
最简单的例子——每天早上 8 点发天气预报：
```
openclaw cron add --name "天气预报" --cron "0 8 * * *" \
  --message "查询今天西安的天气，发送给我" \
  --channel "feishu" \
  --to "<chat_id>"
```

<!--more-->

这个chat_id获取可以从日志里面看到。你在飞书群里面@机器人后发信息过来，就可以在日志里面查看到这个群ID。

飞书channel配置：
```json
{
  "channels": {
    "feishu": {
      "appId": "<appId>",
      "appSecret": "<appSecret>",
      "enabled": true,
      "connectionMode": "websocket",
      "requireMention": true,
      "groupPolicy": "open",
      "groups": {
        "*": {
          "requireMention": true
        }
      },
      "dmPolicy": "pairing"
    }
  }
}
```

三种调度方式按需选：
```
# 固定周期：每 30 分钟执行一次
openclaw cron add --name "定期检查" --every 30m \
  --message "检查有没有需要我处理的事情" \
  --channel "feishu" \
  --to "<chat_id>"

# 一次性延时：20 分钟后提醒
openclaw cron add --name "提醒" --at 20m \
  --message "提醒：该休息一下了！" \
  --channel "feishu" \
  --to "<chat_id>"
```

管理定时任务命令：
```
openclaw cron list                 # 查看所有任务
openclaw cron run <taskId>          # 手动触发一次
openclaw cron runs --id <taskId>    # 查看执行历史
openclaw cron disable <taskId>       # 暂停
openclaw cron enable <taskId>        # 恢复
openclaw cron edit <taskId>           # 编辑
openclaw cron rm <taskId>             # 删除
```

## Workspace文件一览

工作区（Workspace）是龙虾的"家"——它的身份、性格、记忆、技能全都存放在这里。

文件                     | 用途                                   | 加载时机
------------------------|----------------------------------------|----------------------------------------
AGENTS.md               | 操作指令：告诉龙虾"怎么做事"和如何使用记忆    | 每次会话开始
SOUL.md                 | 人设：性格、语气、边界                     | 每次会话开始
USER.md                 | 用户资料：你是谁、怎么称呼你                | 每次会话开始
IDENTITY.md             | 龙虾身份：名字、风格、表情符号              | 每次会话开始
TOOLS.md                | 工具使用备注（不控制工具开关，只是使用建议）   | 每次会话开始
HEARTBEAT.md            | 心跳任务清单                             | 心跳运行时
BOOT.md                 | 启动清单（可选，Gateway 重启时执行）        | Gateway 启动时
BOOTSTRAP.md            | 首次引导仪式（完成后删除）                  | 仅首次引导运行
MEMORY.md               | 长期记忆（可选）                          | 仅主会话
memory/YYYY-MM-DD.md    | 每日记忆日志                             | 每次会话开始：今天+昨天；其余按需读取
skills/                 | 工作区级技能（可选）                       | 按需加载

备注：`BOOT.md` (可选的启动检查清单)：仅在开启内部钩子（internal hooks）后才会在 Gateway 重启时执行。
它是一个完全可选的功能，通常需要用户手动创建或通过设置启用，因此在全新工作区中它不会自动生成。

这些文件在每次对话时都会注入到 AI 模型的上下文窗口中，会消耗 Token。保持文件简洁，尤其是 MEMORY.md——它会随时间增长， 导致上下文使用量增大和更频繁的压缩。
`memory/YYYY-MM-DD.md `中今天和昨天的文件会在会话开始时自动读取，其余日期的文件通过 `memory_search` 和 `memory_get` 工具按需访问，不占用上下文窗口。

先配好身份三件套（让龙虾认识你），再写 AGENTS.md（教它做事），其余按需添加。
每个文件都是纯 Markdown，直接用文本编辑器编辑即可。9 个文件看起来不少，但它们各司其职，可以分为四组来理解：

分组              | 文件                                      | 一句话说明
-----------------|-------------------------------------------|------------------------------
身份三件套         | IDENTITY.md / SOUL.md / USER.md          | 龙虾是谁、什么性格、服务谁
行为指南           | AGENTS.md / TOOLS.md                     | 怎么做事、怎么用工具
记忆系统           | MEMORY.md                                | 跨会话记住什么
生命周期           | BOOTSTRAP.md / BOOT.md / HEARTBEAT.md    | 首次启动 → 每次重启 → 定期巡检


