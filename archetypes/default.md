+++
date = '{{ time.Now.Format "2006-01-02" }}'
draft = true
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
description = ""
tags = [
    "tech",
]
+++
