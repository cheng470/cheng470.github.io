---
title: "使用 Hugo 搭建博客"
date: 2022-08-29T00:21:41+08:00
description: 记录 Hugo 的使用
---

`Hugo` 是由 Go 语言实现的静态网站生成器。这篇文章记录在本地 Arch Linux 上，安装搭建 hugo 的过程，并部署到 github 。

<!--more-->

## 安装

推荐用各自系统的包管理器进行安装。

```bash
pacman -S hugo
```

## 本地创建网站

下面创建一个 `cheng470.github.io` 的网站，然后添加 ananke 主题：

```bash
hugo new site cheng470.github.io
cd cheng470.github.io
git init
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
echo theme = \"ananke\" >> config.toml
```

### 添加文章

```bash
hugo new posts/my-first-post.md
```

### 启动服务

```bash
hugo server -D
```

### 生成静态页面

```bash
hugo -D
```

## 部署到 Github

新建 github 工程 `cheng470.github.io` , 开启 `Pages` 功能，设置分支为 `gh-pages`。

执行如下命令：

```bash
git remote add origin git@github.com:cheng470/cheng470.github.io.git
git pull
```

之后把所有代码提交上去，现在还不能访问，还需要配置 `GitHub Action`，让它帮忙自动运行 hugo 的工具来生成静态页面到 `gh-pages` 分支。

### 配置 GitHub Action

创建文件 `.github/workflows/gh-pages.yml` 内容如下：

```yml
name: github pages

on:
  push:
    branches:
      - main  # Set a branch to deploy
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-20.04
    steps:
      - uses: actions/checkout@v2
        with:
          submodules: true  # Fetch Hugo themes (true OR recursive)
          fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'
          # extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        if: github.ref == 'refs/heads/main'
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
```

提交到仓库中。

这样就可以成功访问了。

## 问题

### 首页的摘要格式很乱

比如：

![摘要格式很乱](/使用-hugo-搭建博客/摘要1乱七八糟.png)

这是因为 hugo 默认取文章的前70个单词作为摘要，内部算法是通过空格对文章内容做 split 后取前面的 70 个单词。

这里是汉字，没有空格，所以看起来很长。可以在 hugo 的配置里面加上 `hasCJKLanguage = true` 配置来修正：

![使用cjk开关后的摘要](/使用-hugo-搭建博客/摘要2使用cjk开关后的效果.png)

但也不好看，hugo 还支持手动生成摘要的方式，在摘要和文章主体内容之间加入摘要分割符 <!\-\-more\-\-> 来生成摘要。效果如下：

![使用摘要分隔符后的摘要](/使用-hugo-搭建博客/摘要3使用摘要分割符后的效果.png)

## 参考

- [Quick Start | Hugo](https://gohugo.io/getting-started/quick-start/)
- [Host on GitHub | Hugo](https://gohugo.io/hosting-and-deployment/hosting-on-github/)
