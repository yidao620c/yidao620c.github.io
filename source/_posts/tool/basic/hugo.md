---
title: 使用Hugo搭建个人站点
date: 2026-04-27 21:56:21
toc: true
categories: [ 开发工具 ]
tags: [ hugo ]
---

Hugo是一个静态网站生成器，基于Go语言，使用Markdown写作。构建速度极快，速度之王 (10000页约2.95秒)。本地写作能实时预览，体验流畅。
单个二进制文件，无需node_modules，升级只需替换新文件，适用于大型内容站，追求极致性能与长期免维护。

官网地址：https://gohugo.io/
<!-- more -->

## 安装

首先安装Go语言：https://go.dev/doc/install

再安装Git：https://git-scm.com/

然后打开Powershell，输入

```
# 先安装 Scoop 包管理器
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
irm get.scoop.sh | iex
```

如果已经是管理员用户了，就输入

```
iex "& {$(irm get.scoop.sh)} -RunAsAdmin"
```

然后安装 Hugo Extended 版（支持 Sass/SCSS 编译）

```
scoop install hugo-extended
scoop install sass
```

## 创建站点并添加主题

```
# 创建新站点
hugo new site xiongneng-net
cd xiongneng-net

# 初始化 Git 仓库
git init

# 以 PaperMod 为例，添加为主题子模块
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

## 最简配置文件

站点根目录下的 hugo.toml（或 config.yaml）是核心配置文件。以下是一个最小可运行配置：

```yaml
# config.yaml（推荐 yaml 格式，更易读）
baseURL: 'https://你的域名.com/'
languageCode: 'zh-CN'
title: '你的站点名称'
theme: 'PaperMod'

params:
  # 主页个人信息
  profileMode:
    enabled: true
    title: "全栈开发者 | Python / Java / Web"
    subtitle: "专注于Web应用开发与系统架构"
    # 头像放在 static/images/ 下
    imageUrl: "images/avatar.jpg"
    # 主页按钮（引导访客到项目页或博客）
    buttons:
      - name: 项目作品
        url: /projects
      - name: 技术博客
        url: /posts

  # 社交链接
  socialIcons:
    - name: github
      url: "https://github.com/你的ID"
    - name: email
      url: "mailto:你的邮箱"
```

创建第一篇内容

```
# 创建文章
hugo new posts/hello-world.md

# 用编辑器打开 content/posts/hello-world.md
# 将 draft: true 改为 draft: false（否则不会发布）
```

启动开发服务器

```
# -D 包含草稿，--bind 0.0.0.0 允许局域网内其他设备预览
hugo server -D --bind 0.0.0.0
```

浏览器访问 `http://localhost:1313`。Hugo 支持热重载——你改动 Markdown 或配置后，浏览器会自动刷新，无需手动重启。

## 文章规范

每篇文章顶部的 Front Matter 不只是给人类看的，它会影响搜索引擎收录效果：

```markdown
---
title: "Hugo 建站完整指南"
date: 2026-04-28
draft: false
tags: ["Hugo", "建站", "静态网站"]
categories: ["教程"]
description: "从零搭建 Hugo 静态博客，并部署到腾讯云的完整教程"
---
```

其中 description 字段会直接渲染为页面的 `<meta name="description">`，对 SEO 至关重要。

## 创建独立页面（作品集、服务页等）

除了博客文章，你还可以创建独立页面来展示项目案例和未来工作室的服务：

```
hugo new projects/_index.md
hugo new services/_index.md
```

每个 `_index.md` 会作为一个独立路由（如 `/projects/`），你可以在对应的 Markdown 中自由排版。

## 生产构建

```
# 生成静态文件到 public/ 目录
hugo --minify

# 文件结构示例：
# public/
# ├── index.html
# ├── posts/
# │   └── hello-world/
# │       └── index.html
# └── ...
```

`--minify` 参数会压缩 HTML/CSS/JS，减小文件体积 30%-50%。`public/` 目录里的所有文件就是你要部署到服务器上的内容。

## GitHub Actions 自动部署

手动 scp 上传虽然可行，但每次写文章都要手动构建+上传，效率不高。利用 GitHub Actions 可以实现「推送代码即自动部署」：

工作流设计思路：在 GitHub Actions 中安装 Hugo → 拉取源码 → 执行 `hugo --minify` 构建 → 通过 `scp` 将 `public/` 上传到腾讯云服务器。

简要步骤：

- 在项目根目录创建 .github/workflows/deploy.yml
- 在 GitHub 仓库 Settings → Secrets 中添加：
    * SERVER_IP：腾讯云服务器 IP
    * SERVER_USER：SSH 用户名
    * SERVER_PASSWORD：SSH 密码
- 每次 git push 到 main 分支时自动触发构建和部署

更多部署选项（包括 Netlify 等免费托管方案）可参考 [Hugo 官方部署文档](https://gohugo.io/hosting-and-deployment/)。

## 日常维护命令手册

 场景              | 命令                                    
-----------------|---------------------------------------
 新建文章            | hugo new posts/文章标题.md                
 本地预览（含草稿）       | hugo server -D                        
 生产构建            | hugo --minify                         
 清理 public 目录后重建 | rm -rf public/ && hugo --minify       
 更新主题子模块         | git submodule update --remote --merge 
 检查 Hugo 版本      | hugo version                          

## 参考
* Hugo 官方文档：[gohugo.io/documentation](https://gohugo.io/documentation/)
* Hugo 主题市场：[themes.gohugo.io](https://themes.gohugo.io/)（300+ 免费主题）
* 代码托管：[GitHub](https://github.com/)（存储 Hugo 源码 + 配合 Actions 自动部署）
* GitHub Pages 免费托管：[gohugo.io/host-on-github-pages](https://gohugo.io/host-and-deploy/host-on-github-pages/)

