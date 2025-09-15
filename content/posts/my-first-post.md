+++
date = '2025-09-16T00:07:09+08:00'
draft = true
title = '使用 Hugo 搭建博客'
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

新建 github 工程 `cheng470.github.io` , 开启 `Pages` 功能，设置分支为 `gh-pages`。

执行如下命令：

```bash
git remote add origin git@github.com:cheng470/cheng470.github.io.git
git pull
```

之后把所有代码提交上去，现在还不能访问，还需要配置 `GitHub Action`，让它帮忙自动运行 hugo 的工具来生成静态页面到 `gh-pages` 分支。
