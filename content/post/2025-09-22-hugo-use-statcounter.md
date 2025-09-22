+++
date = '2025-09-22'
draft = false
title = 'Hugo 使用 Statcounter'
description = ""
tags = [
    "hugo",
    "tech",
]
+++

## 生成模板代码

1. 访问 [Statcounter Projects](https://statcounter.com/) 官网
1. 注册并新建 Project，使用免费的即可
1. 配置样式，我这里去掉所有样式
1. 生成模板代码，代码类似于

```html
  <!-- Default Statcounter code for cheng470 的博客 https://cheng470.github.io/-->
  <script type="text/javascript">
  ...
  </script>
  <noscript>
  ...
  </noscript>
  <!-- End of Statcounter Code -->
```

## 将代码放到hugo

新建文件 `layouts/partials/footer.html`，这里打算覆盖主题paper的相同文件，所以先把主题的 `footer.thml` 文件复制过来，然后添加 `Statcounter` 的模板代码，完整的代码类似于：

```html
<footer
  class="mx-auto flex h-[4.5rem] max-w-(--w) items-center px-8 text-xs tracking-wider uppercase opacity-60"
>
  <div class="mr-auto">
    {{- if site.Copyright -}} {{- site.Copyright -}} {{- else -}} &copy; {{-
    now.Year }}
    <a class="link" href="{{- `` | absURL -}}">{{- site.Title -}}</a>
    {{- end -}}
  </div>
  <a class="link mx-6" href="https://gohugo.io/" rel="noopener" target="_blank"
    >powered by hugo️️</a
  >️
  <a
    class="link"
    href="https://github.com/nanxiaobei/hugo-paper"
    rel="noopener"
    target="_blank"
    >hugo-paper</a
  >
  <!-- Default Statcounter code for cheng470 的博客 https://cheng470.github.io/-->
  <span>pv:</span>
  <script type="text/javascript">
    ...
  </script>
  <noscript>
    ...
  </noscript>
  <!-- End of Statcounter Code -->
</footer>
```

上传到 github 重新部署，看看是否生效。
