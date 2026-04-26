---
title: Ubuntu24安装小龙虾OpenClaw
date: 2026-03-31 21:53:19 +0800
toc: true
categories: [ AI ]
tags: [ OpenClaw ]
---

之前在Windows本地机器安装过OpenClaw，但是不能长期稳定运行，因此最后选择云主机。我选择的是腾讯云主机。配置如下

```
轻量应用服务器新购	
运算组件:  2核CPU、8GB内存 (锐驰型Linux-2核8G-80G-200Mbps峰值带宽)
系统盘:    80GB SSD云硬盘
峰值带宽:  200Mbps峰值带宽
地域:     中国香港
镜像:     Ubuntu 24.04 LTS
```

购买地址：https://cloud.tencent.com/product/lighthouse

LLM模型购买的套餐MiniMax Max，后台使用模型为MiniMax-M2.7。

<!--more-->

## 安装脚本+微信配置

注意：下面的API_KEY是Token Plan的那个KEY啊，不是普通的那个API_KEY。

```
#!/bin/bash
# 一键安装 OpenClaw + MiniMax-M2.7 + 微信（ubuntu 用户专用）
# 替换成你的 MiniMax API Key
API_KEY="你的MiniMax_API_KEY"
# 1. 安装依赖
sudo apt update -y && sudo apt install -y qrencode
# 2. 静默安装 OpenClaw
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --yes
# 3. 刷新环境
source ~/.bashrc
echo 'export PATH="$HOME/.openclaw/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
# 4. 配置 MiniMax-M2.7（香港服务器专用 CN 接口）
openclaw config set llm.provider "minimax-cn"
openclaw config set llm.apiKey "${API_KEY}"
openclaw config set llm.model "MiniMax-M2.7"
openclaw config set llm.maxTokens 4096
# 5. 重启网关
openclaw gateway restart
```

## 配置微信接入

```
# 1. 安装微信官方插件
openclaw plugins install @tencent-weixin/openclaw-weixin --yes
# 2. 启用微信通道
openclaw config set plugins.entries.openclaw-weixin.enabled true
# 3. 直接生成微信绑定二维码（无弹窗）
openclaw channels login --channel openclaw-weixin --no-prompt
# 5. 重启网关
openclaw gateway restart

# 重新登录微信（出二维码）
openclaw channels login --channel openclaw-weixin --no-prompt
```

然后在手机上面打开微信的openclaw插件，扫描这个二维码即可完成配对。

### 删除openclaw-weixin
如果不想使用微信接入了，可以删除掉。
```
# 1. 先停止网关，防止操作过程中状态不一致
openclaw gateway stop
# 2. 在配置中禁用微信通道
openclaw config set channels.openclaw-weixin.enabled false

# 方法一：优先尝试使用官方命令卸载
openclaw plugins uninstall openclaw-weixin
# 方法二：如果官方命令无效，手动删除插件目录
rm -rf ~/.openclaw/extensions/openclaw-weixin

# 1. 重启网关
openclaw gateway restart
# 2. 检查服务状态，确认微信通道已不在列表中或显示为 OFF
openclaw status
```

## 配置飞书接入

### 飞书平台操作步骤

1、登录飞书开放平台：访问飞书开放平台，登录你的飞书账号。
2、创建企业自建应用：点击“创建企业自建应用”，填写名称（如“OpenClaw 助手”）和描述后点击“创建”。
3、获取应用凭证（App ID 和 App Secret）：

- 进入应用管理页，在左侧菜单找到“凭证与基础信息”。
- 在此页面找到“App ID”和“App Secret”，点击复制并妥善保存。
  4、添加机器人能力：在左侧菜单点击“添加应用能力”，找到并添加“机器人”能力。
  5、配置应用权限：在左侧菜单进入“权限管理”，点击“批量导入/导出权限”，选择“应用身份权限”后将下方 JSON 代码粘贴进去，点击“申请开通”

```json
{
  "scopes": {
    "tenant": [
      "application:application:self_manage",
      "application:bot.menu:write",
      "aily:file:read",
      "aily:file:write",
      "cardkit:card:read",
      "cardkit:card:write",
      "contact:contact.base:readonly",
      "contact:user.base:readonly",
      "contact:user.employee_id:readonly",
      "corehr:file:download",
      "docs:document.comment:create",
      "docs:document.comment:delete",
      "docs:document.comment:read",
      "docs:document.comment:update",
      "docs:document.comment:write_only",
      "docx:document.block:convert",
      "docx:document:create",
      "docx:document:readonly",
      "docx:document:write_only",
      "docs:document.content:read",
      "drive:drive.metadata:readonly",
      "drive:drive.search:readonly",
      "search:docs:read",
      "sheets:spreadsheet",
      "wiki:wiki:readonly",
      "im:chat",
      "im:chat:read",
      "im:chat:readonly",
      "im:chat:update",
      "im:chat.members:bot_access",
      "im:message",
      "im:message.group_at_msg:readonly",
      "im:message.group_msg",
      "im:message.p2p_msg:readonly",
      "im:message.pins:read",
      "im:message.pins:write_only",
      "im:message.reactions:read",
      "im:message.reactions:write_only",
      "im:message:readonly",
      "im:message:recall",
      "im:message:send_as_bot",
      "im:message:send_multi_users",
      "im:message:send_sys_msg",
      "im:message:update",
      "im:resource"
    ],
    "user": [
      "contact:user.employee_id:readonly",
      "offline_access",
      "contact:contact.base:readonly",
      "aily:file:read",
      "aily:file:write",
      "im:chat.access_event.bot_p2p_chat:read"
    ]
  }
}
```

6、配置事件订阅：

- 事件与回调的配置，都设置成`使用长连接接收事件`。
- 然后在事件配置，这里点击添加事件，增加下面的几个订阅事件。
- 在版本管理与发布中，把这个新的应用发布上线。

订阅事件说明：

 事件                            | 说明       
-------------------------------|----------
 im.message.receive_v1         | 接受消息（必须） 
 im.message.message_read_v1    | 消息已读回执   
 im.chat.member.bot.added_v1   | 机器人进群    
 im.chat.member.bot.deleted_v1 | 机器人被移出群  

### 在 OpenClaw 中配置飞书（纯命令行操作）

回到你的 Ubuntu 服务器，通过 SSH 连接并执行以下命令。

```
#安装飞书插件
openclaw plugins install feishu-openclaw
#配置飞书通道的 App ID 和 Secret
openclaw config set channels.feishu.appId "cli_xxxxx"
openclaw config set channels.feishu.appSecret "your_app_secret"
#设置核心配置项：继续执行以下命令，确保服务能正确接收并响应消息。
# 1. 启用飞书通道
openclaw config set channels.feishu.enabled true
# 2. 推荐使用 websocket 长连接模式
openclaw config set channels.feishu.connectionMode websocket
# 3. 开启私聊（直接消息），新私聊需配对，保证安全
openclaw config set channels.feishu.dmPolicy pairing
# 4. 只在群聊中被 @ 时才会响应，避免误触发
openclaw config set channels.feishu.requireMention true
#重启 OpenClaw 网关
openclaw gateway restart
#检查服务状态：验证飞书通道是否正常运行。
openclaw status
```

### 开始配对

1、在飞书 App 中搜索你创建的应用名称（例如“OpenClaw助手”）。
2、进入聊天窗口，发送“你好”等测试消息。机器人会给你回复一个配对指令。
3、然后在命令行执行上面的配对指令：`openclaw pairing approve feishu <配对码>` 完成配对

## 后续更新配置

```
openclaw configure
```

## 使用说明

- 把脚本中 你的MiniMax_API_KEY 替换成你真实的密钥，格式：sk-xxxxxxx
- 用默认用户 ubuntu 登录服务器
- 整段粘贴 → 回车运行
- 全程全自动，最后直接出二维码
- 打开：微信 → 我 → 设置 → 插件 → ClawBot → 扫一扫。绑定成功即可使用 AI 对话

## 一键开启防火墙（只开 22，其他全关）

微信连接完全不受影响，服务器彻底安全。

```
# 重置防火墙
sudo ufw reset
# 只允许 SSH（22 端口），其他全部拒绝
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
# 启用防火墙
sudo ufw enable
# 查看最终状态
sudo ufw status
ubuntu@VM-0-3-ubuntu:~$ sudo ufw status
Status: active
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere                  
22/tcp (v6)                ALLOW       Anywhere (v6) 
```

## 管理 OpenClaw

完全不用网页后台，这样管理 OpenClaw

1、查看配置（命令行）

```
# 查看全部配置
openclaw config get
# 查看某一项（比如模型）
openclaw config get llm
openclaw config get llm.apiKey
openclaw config get channels.openclaw-weixin
```

修改配置（命令行，推荐）

```
# 改模型密钥
openclaw config set llm.apiKey "sk-xxxxxx"
# 改模型
openclaw config set llm.model "MiniMax-M2.7"
# 改最大 token
openclaw config set llm.maxTokens 4096
# 改完重启网关
openclaw gateway restart --yes
```

直接编辑配置文件（进阶）

```
# 备份（好习惯）
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
# 编辑
nano ~/.openclaw/openclaw.json
# 保存退出：Ctrl+O → 回车 → Ctrl+X
# 重启网关
openclaw gateway restart
```

常用命令（你以后全用这些）

```
# 查看状态
openclaw status
openclaw gateway status
# 重启
openclaw gateway restart
# 查看日志
openclaw logs --follow
```

## 安装为系统服务

默认安装完成后openclaw网关并没有自动安装为系统服务。可执行

```
sudo tee /etc/systemd/system/openclaw.service > /dev/null << 'EOF'
[Unit]
Description=OpenClaw AI Gateway
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu
ExecStart=/home/ubuntu/.openclaw/bin/openclaw gateway start --no-prompt
Restart=always
RestartSec=5
Environment="PATH=/home/ubuntu/.openclaw/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin"

[Install]
WantedBy=multi-user.target
EOF
```

然后执行：

```
# 重新加载服务配置
sudo systemctl daemon-reload
# 开启开机自启
sudo systemctl enable openclaw
# 启动服务
sudo systemctl start openclaw
# 查看状态
sudo systemctl status openclaw
#查看系统所有服务，过滤看看有没有 openclaw 相关自启
systemctl list-unit-files --type=service | grep -E 'openclaw|claw'
```

## 执行exec时报权限不足需要pair

我让它直接帮我开通80和443的防火墙，需要用到sudo ufw命令。报错：

```
这个还是要用到 exec 工具，但现在还是 pairing required 的问题没解决，没法直接跑命令。
```

先执行下面这条命令，找到Pending状态的一个Request，记住这个RequestId。

```
openclaw devices list
openclaw devices approve $RequestId
openclaw gateway restart
```

或者最简单方式就是完全信任exec工具：

```
openclaw config set tools.exec.security full
openclaw gateway restart
```

## OpenClaw版本升级
```
openclaw update --channel stable
```

## 迁移到Docker容器
如果在云主机上面安装OpenClaw，最安全和推荐的做法是使用Docke容器安装。

**第一步：安装docker，拉取镜像。**

因为我的是香港主机，因此直接使用官方源。
```
# 1. 安装 Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker  # 或者直接重新登录 SSH

# 2. 拉取 OpenClaw 官方镜像
docker pull ghcr.io/openclaw/openclaw:latest
```

如果是国内主机，则使用下面的命令
```
# 1. 安装 Docker（使用国内源，推荐）
sudo apt-get remove -y docker docker-engine docker.io containerd runc
sudo apt-get update && sudo apt-get install -y ca-certificates curl gnupg lsb-release
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/trusted.gpg.d/docker.gpg] http://mirrors.aliyun.com/docker-ce/linux/ubuntu noble stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker run --rm hello-world

# 2. 配置免 sudo 权限（可选）
sudo usermod -aG docker $USER && newgrp docker

# 3. 配置国内镜像加速（强烈建议）
sudo tee /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]
}
EOF
sudo systemctl daemon-reload && sudo systemctl restart docker

# 4. 拉取 OpenClaw 镜像（国内源）
docker pull registry.cn-hangzhou.aliyuncs.com/qiluo-images/openclaw:latest
docker tag registry.cn-hangzhou.aliyuncs.com/qiluo-images/openclaw:latest openclaw:latest

```

**第二步：备份宿主机上面的openclaw配置**

这是最重要的一步。`~/.openclaw` 目录存储了所有配置、认证、已安装技能和会话历史。我们把它完整打包，为迁移做准备。
命令执行后，你会得到一个名为 `openclaw-backup.tar.gz` 的备份文件，它包含了你的全部数据
```
cd ~
# 先停止旧版服务，防止迁移时写入数据
openclaw gateway stop
# 清理日志以减小备份包体积
rm -rf ~/.openclaw/logs/*.log

# 打包整个配置目录（注意排除 logs 和 node_modules 等不必要的大文件夹）
tar -czvf openclaw-backup.tar.gz \
  --exclude='.openclaw/logs' \
  .openclaw
```

**第三步：映射数据目录**
```
# 创建数据目录
sudo mkdir -p /data/openclaw
sudo tar -xzvf openclaw-backup.tar.gz -C /data/openclaw --strip-components=1
sudo chown -R 1000:1000 /data/openclaw
sudo chmod -R u+rwX /data/openclaw
```

**第四步：使用 Docker 命令启动容器**
```
docker run -d \
  --name openclaw \
  --restart unless-stopped \
  -p 18789:18789 \
  -v /data/openclaw:/home/node/.openclaw \
  ghcr.io/openclaw/openclaw:latest
```

**第五步：验证迁移状态**
```
docker logs openclaw
docker exec -it openclaw openclaw doctor
docker exec -it openclaw openclaw status
```
在浏览器中打开 `http://你的服务器IP:18789`，如果能正常访问，就说明迁移基本成功了。

**第六步：6. 清理旧环境（可选）**

在确认 Docker 版本的 OpenClaw 工作完全正常后，你就可以安全地移除直接安装的版本了。
```
# 停止并禁用旧版 systemd 服务（如果存在）
sudo systemctl stop openclaw
sudo systemctl disable openclaw

# 卸载openclaw软件
openclaw uninstall --all --yes --non-interactive
npm uninstall -g openclaw
# 直接删除旧版的 .openclaw 目录
rm -rf ~/.openclaw
```

