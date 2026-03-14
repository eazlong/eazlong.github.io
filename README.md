# 🚀 My Hugo Blog

基于 [Hugo](https://gohugo.io/) + [Stack 主题](https://github.com/CaiJimmy/hugo-theme-stack) 搭建的个人博客。

## ✨ 功能特性

- 📝 技术博客 / 文章系统
- 🎨 自动跟随系统明暗模式
- 🔍 全文搜索
- 📦 项目展示页面
- 👤 关于我页面
- 📊 归档 / 标签 / 分类
- 📡 RSS Feed
- ⚡ 极速构建（Hugo）

## 🛠 本地开发

### 前置要求

- Hugo Extended v0.121.0+（必须是 Extended 版本）
- Git

### 安装 Hugo

```bash
# macOS
brew install hugo

# Windows (Scoop)
scoop install hugo-extended

# Windows (Chocolatey)
choco install hugo-extended

# Linux (Snap)
snap install hugo

# Linux (直接下载)
# 从 https://github.com/gohugoio/hugo/releases 下载 hugo_extended_*_linux-amd64.tar.gz
```

### 克隆并启动

```bash
# 1. 克隆仓库（含子模块）
git clone --recurse-submodules https://github.com/yourname/my-blog.git
cd my-blog

# 2. 如果已经克隆但没有拉取子模块
git submodule update --init --recursive

# 3. 本地启动（带草稿）
hugo server -D

# 访问 http://localhost:1313
```

## 📝 写新文章

```bash
# 创建新文章（推荐使用 Page Bundle 结构）
hugo new post/文章标题/index.md
```

文章 Front Matter 模板：

```yaml
---
title: "文章标题"
description: "文章简介，用于 SEO 和卡片预览"
date: 2024-01-01
lastmod: 2024-01-01
image: "cover.jpg"          # 封面图（放在同目录下）
categories:
    - 技术文章
tags:
    - Go
    - 后端
draft: false                # true = 草稿，不会发布
---
```

## 📁 目录结构

```
my-blog/
├── hugo.yaml                 # 主配置文件 ← 主要在这里修改
├── content/
│   ├── post/                 # 博客文章
│   │   └── 文章名/
│   │       ├── index.md      # 文章正文
│   │       └── cover.jpg     # 封面图（可选）
│   ├── about/
│   │   └── index.md          # 关于我页面
│   ├── projects/
│   │   └── index.md          # 项目展示页面
│   └── archives/
│       └── index.md          # 归档页面（无需编辑）
├── assets/
│   └── scss/
│       └── custom.scss       # 自定义样式
├── static/
│   └── img/
│       └── avatar.png        # 头像（替换为你的）
├── themes/
│   └── hugo-theme-stack/     # 主题（git submodule）
└── .github/
    └── workflows/
        └── hugo.yaml         # GitHub Actions 自动部署
```

## 🎨 个性化配置

编辑 `hugo.yaml` 中的以下字段：

```yaml
title: "你的名字"                          # 网站标题
baseURL: "https://yourname.github.io/"    # 你的域名

params:
  sidebar:
    emoji: 🚀                              # 侧边栏 Emoji
    subtitle: "你的个人简介"

  footer:
    since: 2024                            # 建站年份
    customText: "你的个人签名"
```

## 🚀 部署到 GitHub Pages

1. 在 GitHub 创建仓库（名字为 `yourname.github.io` 或普通仓库名）

2. 修改 `hugo.yaml` 中的 `baseURL` 为你的 GitHub Pages 地址

3. 推送代码：

```bash
git add .
git commit -m "initial commit"
git remote add origin https://github.com/yourname/my-blog.git
git push -u origin main
```

4. 在仓库 Settings → Pages → Source 选择 **GitHub Actions**

5. 之后每次 `push` 到 `main` 分支，GitHub Actions 会自动构建并部署 🎉

## 📖 参考资源

- [Hugo 官方文档](https://gohugo.io/documentation/)
- [Stack 主题文档](https://stack.jimmycai.com/)
- [Stack 主题 GitHub](https://github.com/CaiJimmy/hugo-theme-stack)
