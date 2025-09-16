+++
author = "cheng470"
title = "使用 Hugo 搭建博客"
date = "2025-09-16"
description = "记录自己使用 Hugo 搭建博客的过程"
tags = [
    "tech",
    "hugo",
]
weight = 10
+++

## 安装

推荐用各自系统的包管理器进行安装。

```bash
pacman -S hugo
```

查看版本：

```sh
$ hugo version
hugo v0.150.0+extended+withdeploy linux/amd64 BuildDate=unknown
```

## 本地创建网站

下面创建一个 `cheng470.github.io` 的网站，然后添加 ananke 主题：

```bash
hugo new site cheng470.github.io
cd cheng470.github.io
git init
git submodule add https://github.com/theNewDynamic/gohugo-theme-ananke.git themes/ananke
echo theme = \"ananke\" >> config.toml
hugo server
```

> 这里用到了 git 子模块功能，如果在其他电脑 clone 本仓库，需要执行 `git submodule update --init --recursive` 把子模块代码也拉下来。 

### 添加文章

```bash
hugo new content content/posts/my-first-post.md
```

打开生成的 `my-first-post.md`，可以看到如下内容：

```md
+++
title = 'My First Post'
date = 2024-01-14T07:07:07+01:00
draft = true
+++
```

默认生成的文章是草稿状态，需要改成 false,才能发布。

使用如下命令之一，就可以预览草稿文章：

```sh
hugo server --buildDrafts
hugo server -D
```

## 配置网站

打开 `hugo.toml`，修改如下：

```toml
baseURL = 'https://cheng470.github.io/'
languageCode = 'en-us'
title = 'cheng470 的博客'
theme = 'ananke'
```

之后运行 `hugo server -D` 进行预览。

## 部署到 Github

### 1 创建仓库并配置 GitHub Action

新建 github 工程 `cheng470.github.io` , 开启 `Pages` 功能，设置分支为 `GitHub Actions`。

执行如下命令：

```bash
git remote add origin git@github.com:cheng470/cheng470.github.io.git
git pull
```

之后把所有代码提交上去，现在还不能访问，还需要配置 `GitHub Action`，让它帮忙自动运行 hugo 的工具来生成静态页面到 `gh-pages` 分支。

### 2 创建 github workflow

更新 `hugo.toml` 文件，配置图片缓存目录：

```toml
[caches]
  [caches.images]
    dir = ':cacheDir/images'
```

创建 github workflow 文件：

```sh
mkdir -p .github/workflows
touch .github/workflows/hugo.yaml
```

该文件添加如下内容：

```yaml
name: Build and deploy
on:
  push:
    branches:
      - main
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: false
defaults:
  run:
    shell: bash
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      DART_SASS_VERSION: 1.92.1
      GO_VERSION: 1.25.1
      HUGO_VERSION: 0.150.0
      NODE_VERSION: 22.18.0
      TZ: Europe/Oslo
    steps:
      - name: Checkout
        uses: actions/checkout@v5
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: false
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Create directory for user-specific executable files
        run: |
          mkdir -p "${HOME}/.local"
      - name: Install Dart Sass
        run: |
          curl -sLJO "https://github.com/sass/dart-sass/releases/download/${DART_SASS_VERSION}/dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          tar -C "${HOME}/.local" -xf "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          rm "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          echo "${HOME}/.local/dart-sass" >> "${GITHUB_PATH}"
      - name: Install Hugo
        run: |
          curl -sLJO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir "${HOME}/.local/hugo"
          tar -C "${HOME}/.local/hugo" -xf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          rm "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "${HOME}/.local/hugo" >> "${GITHUB_PATH}"
      - name: Verify installations
        run: |
          echo "Dart Sass: $(sass --version)"
          echo "Go: $(go version)"
          echo "Hugo: $(hugo version)"
          echo "Node.js: $(node --version)"
      - name: Install Node.js dependencies
        run: |
          [[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true
      - name: Configure Git
        run: |
          git config core.quotepath false
      - name: Cache restore
        id: cache-restore
        uses: actions/cache/restore@v4
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: hugo-${{ github.run_id }}
          restore-keys:
            hugo-
      - name: Build the site
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --cacheDir "${{ runner.temp }}/hugo_cache"
      - name: Cache save
        id: cache-save
        uses: actions/cache/save@v4
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: ${{ steps.cache-restore.outputs.cache-primary-key }}
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

将改动推送到 GitHub 仓库。

### 3 查看博客页面

访问 <https://github.com/cheng470/cheng470.github.io/actions> 页面，查看 workflow 执行状态。

等待执行成功后，访问 `cheng470.github.io` 查看博客页面。

## 更换主题 hugo-paper

看到网上的推荐 [nanxiaobei/hugo-paper: 🪺 A simple, clean, customizable Hugo theme](https://github.com/nanxiaobei/hugo-paper)

感觉不错，换一下：

```sh
git submodule add https://github.com/nanxiaobei/hugo-paper themes/paper
```

修改 hugo.toml 配置文件：

```toml
baseURL = 'https://cheng470.github.io/'
languageCode = 'en-us'
title = 'cheng470 的博客'
theme = 'paper'

[caches]
  [caches.images]
    dir = ':cacheDir/images'

[markup]
  [markup.goldmark]
    [markup.goldmark.renderer]
      unsafe = true

[params]
  # color style
  color = 'linen' # linen, wheat, gray, light

  # header social icons
  github = 'cheng470'

  # home page profile
  avatar = 'https://avatars.githubusercontent.com/u/2941980'
  name = 'cheng470'
  bio = 'keep learning'
```

## 参考

- [Quick start](https://gohugo.io/getting-started/quick-start/)
- [Host on GitHub Pages](https://gohugo.io/host-and-deploy/host-on-github-pages/)
