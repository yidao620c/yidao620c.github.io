---
title: Ubuntu24安装小龙虾OpenClaw
date: 2026-03-31 21:53:19 +0800
toc: true
categories: [ AI编程 ]
tags: [ OpenClaw ]
---

之前在Windows本地机器安装过OpenClaw，但是不能长期稳定运行，因此最后选择云主机。我选择的是腾讯云主机。配置如下
```
轻量应用服务器新购	
运算组件:  2核CPU、8GB内存 (锐驰型Linux-2核8G-80G-200Mbps峰值带宽)
系统盘:    80GB SSD云硬盘 (锐驰型Linux-2核8G-80G-200Mbps峰值带宽)
峰值带宽:  200Mbps峰值带宽 (锐驰型Linux-2核8G-80G-200Mbps峰值带宽)
地域:     中国香港
镜像:     Ubuntu 24.04 LTS
```

购买地址：https://cloud.tencent.com/product/lighthouse

LLM模型购买的套餐MiniMax Max，后台使用模型为MiniMax-M2.7。

<!--more-->

## 安装脚步
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
# 4. 安装微信官方插件
openclaw plugins install @tencent-weixin/openclaw-weixin --yes
# 5. 启用微信通道
openclaw config set plugins.entries.openclaw-weixin.enabled true
# 6. 配置 MiniMax-M2.7（香港服务器专用 CN 接口）
openclaw config set llm.provider "minimax-cn"
openclaw config set llm.apiKey "${API_KEY}"
openclaw config set llm.model "MiniMax-M2.7"
openclaw config set llm.maxTokens 4096
# 7. 重启网关
openclaw gateway restart --yes
# 8. 直接生成微信绑定二维码（无弹窗）
sleep 3
openclaw channels login --channel openclaw-weixin --no-prompt
```

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
openclaw gateway restart --yes
```

常用命令（你以后全用这些）
```
# 查看状态
openclaw status
openclaw gateway status
# 重启
openclaw gateway restart --yes
# 重新登录微信（出二维码）
openclaw channels login --channel openclaw-weixin --no-prompt
# 查看日志
openclaw logs -f
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


