---
title: "用 Hugo + Stack 主题搭建个人博客"
description: "从零开始，一步步构建一个支持自动明暗模式、SEO 友好的个人博客"
date: 2020-02-20
lastmod: 2024-02-20
categories:
    - 工具与效率
tags:
    - Hugo
    - 博客搭建
    - GitHub Pages
draft: false
---

本文记录我使用 Hugo 和 Stack 主题搭建个人博客的完整过程。

## 为什么选择 Hugo？

相比 Hexo、Jekyll 等静态博客框架，Hugo 有以下优势：

- ⚡ **极快的构建速度** — 毫秒级渲染，即使文章数量很多
- 🦕 **单二进制文件** — 无需 Node.js/Ruby 环境，安装极简
- 📦 **强大的模板系统** — 高度可定制
- 🔥 **实时预览** — `hugo server` 支持热重载

## 为什么选择 Stack 主题？

Stack 主题专为内容创作者设计，功能完善：

- 卡片式布局，视觉效果好
- 自动跟随系统明暗模式
- 内置搜索、归档、标签云
- 支持数学公式、代码高亮
- 完善的 SEO 支持

## 安装步骤

### 1. 安装 Hugo

```bash
# macOS
brew install hugo

# Windows (使用 Scoop)
scoop install hugo-extended

# Linux
snap install hugo
```

验证安装：

```bash
hugo version
# hugo v0.121.0+extended ...
```

### 2. 创建新站点

```bash
hugo new site my-blog
cd my-blog
git init
```

### 3. 安装 Stack 主题

```bash
git submodule add \
  https://github.com/CaiJimmy/hugo-theme-stack.git \
  themes/hugo-theme-stack
```

### 4. 配置站点

将 `themes/hugo-theme-stack/exampleSite/` 下的配置文件复制到项目根目录，根据自己需求修改 `hugo.yaml`。

### 5. 创建第一篇文章

```bash
hugo new post/my-first-post/index.md
```

### 6. 本地预览

```bash
hugo server -D
# 访问 http://localhost:1313
```

## 部署到 GitHub Pages

创建 `.github/workflows/hugo.yaml`：

```yaml
name: Deploy Hugo Site

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true
      
      - uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true
      
      - run: hugo --minify
      
      - uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

推送代码后，GitHub Actions 会自动构建并部署到 `gh-pages` 分支。

## 小结

整个搭建过程不到 30 分钟，Hugo + Stack 的组合能让你专注于内容创作，而不是折腾配置。享受写作吧！
