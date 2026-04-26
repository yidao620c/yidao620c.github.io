---
title: Windows11安装小龙虾OpenClaw
date: 2026-02-01 20:52:16 +0800
toc: true
categories: [ AI ]
tags: [ OpenClaw ]
---

安装NodeJS后，配置国内源码。

```
npm config set strict-ssl false
npm config set registry https://registry.npmmirror.com
git config --global url."https://github.com/".insteadOf "ssh://git@github.com/"
```

还要开启代理，因为安装过程要访问Github。然后再输入如下命令安装：

```
npm install -g openclaw@latest
```

按理说，输入下面的命令等一个 2 分钟，你会看到下面这种提示：已经添加了 500 多包。

<!-- more -->

再输入如下命令：

```
openclaw onboard --install-daemon
```

大概 1 分钟就能看到下面这个界面。

```
I understand this is personal-by-default and shared/multi-user use requires lock-down. Continue?
```

选择Yes，继续选择QuickStart模式。

② 配置大模型 API

接下来配置Model/auth provider，这一步是需要设置你的大模型 API，智谱、Kimi、MiniMax、Qwen 或者国外的都行。
要区分国内还是国外的，一般国内会有一个 cn 的后缀。我这里选择的是`MiniMax CN - API Key（minimax.com）`，输入APIKEY，
模型名称输入`minimax/MiniMax-M2.7`。默认这个模型没有在列表中，通过手动输入。

③ 选择一个 channel

我先接入到飞书里面吧。下面有创建飞书应用的步骤。创建完成后填写飞书的信息，这里Group chat policy选择Open - respond in all
groups

## 创建飞书应用

前提是先去飞书开放平台创建一个应用。地址：https://open.feishu.cn/app
显示配置权限管理，批量导入权限。自己看情况。。。

```
{
  "scopes": {
    "tenant": [
      "aily:file:read",
      "aily:file:write",
      "cardkit:card:write",
      "contact:contact.base:readonly",
      "contact:user.base:readonly",
      "im:chat",
      "im:chat:read",
      "im:chat:readonly",
      "im:message",
      "im:message.group_at_msg:readonly",
      "im:message.group_msg",
      "im:message.p2p_msg:readonly",
      "im:message.reactions:read",
      "im:message:readonly",
      "im:message:send_as_bot",
      "im:message:update",
      "im:resource"
    ],
    "user": [
      "contact:contact.base:readonly"
    ]
  }
}
```

需要配置的地方：

- 事件与回调的配置，都设置成长链接的方式。
- 然后再事件配置，这里点击添加事件，增加我这里类似的事件。
- 机器人进群、机器人被移出群、消息已读、接受消息。
- 然后去复制你的 APP ID 和密钥。再回到终端命令行，把相关的 ID 和密钥都复制进去就行了。
- 在版本管理与发布中，把这个新的应用发布上线。

订阅事件说明：

 事件                            | 说明       
-------------------------------|----------
 im.message.receive_v1         | 接受消息（必须） 
 im.message.message_read_v1    | 消息已读回执   
 im.chat.member.bot.added_v1   | 机器人进群    
 im.chat.member.bot.deleted_v1 | 机器人被移出群  

注意：订阅事件的操作一定要在接入飞书channel之后，否则选长连接会失败。

④其他配置
接下来命令行中会问你要不要配置搜索、SKill、Hook 等。我建议是都先跳过，然后就可以在 Web 中打开 Gateway 管理模板了。

⑤ How do you want to hatch your bot?
选择`Open the Web UI`界面打开。打开地址：http://127.0.0.1:18789/
后面也可以通过 `openclaw dashboard` 命令来打开。

⑥ 完成配对
进入飞书APP，会看到刚刚发布的应用"小龙虾"。然后你给他发一个消息"你好呀"。它会回复一段文字。把他们复制到 Web 端的聊天框中。
最后等openclaw回复确认，你不用自己去处理，而是回复消息"帮我处理"让小龙虾自己帮你处理。配对成功，然后飞书中就能和OpenClaw聊天了。

## 修改模型配置

我前面安装配置的时候选择模型都是MiniMax-M2.7，但是安装完后查看配置文件还是有2.5配置。因此我直接让OpenClaw帮我修改

```
帮我把所有MiniMax模型配置都改成M2.7，主配置也改成M2.7，不要有M2.5配置。
```

## 开机自动启动

默认情况下OpenClaw的gateway会开机自启动。如果不想让它自动启。则Win+R，输入taskschd.msc。
在任务计划程序列表中找到OpenClaw Gateway，右键选择禁用即可。

## 卸载

本地Windows上面只是尝尝鲜，真正长期稳定养虾，建议使用阿里云主机。本地卸载命令

```powershell
Set-ExecutionPolicy RemoteSigned
openclaw uninstall --all --yes
npm uninstall -g openclaw
```

重启电脑：完成上述步骤后，建议重启一次，确保所有后台进程彻底关闭
