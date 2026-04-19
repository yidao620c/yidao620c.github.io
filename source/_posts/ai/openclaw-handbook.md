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
  --message "查询今天的天气，发送给我" \
  --channel "feishu:chat:你的OpenID"
```

<!--more-->

这个OpenID获取可以从日志里面看到。你用飞书发信息过来，会有一个ou_e6826xxx的随机字符串。

三种调度方式按需选：
```
# 固定周期：每 30 分钟执行一次
openclaw cron add --name "定期检查" --every 30m \
  --message "检查有没有需要我处理的事情"

# 一次性延时：20 分钟后提醒
openclaw cron add --name "提醒" --at 20m \
  --message "提醒：该休息一下了！"
```

管理定时任务命令：
```
openclaw cron list                 # 查看所有任务
openclaw cron run 天气预报           # 手动触发一次
openclaw cron runs --id <taskID>    # 查看执行历史
openclaw cron disable 天气预报       # 暂停
openclaw cron enable 天气预报        # 恢复
openclaw cron edit <jobId>           # 编辑
openclaw cron rm <jobId>             # 删除
```
