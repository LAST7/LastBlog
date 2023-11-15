---
title: 启用 Gitalk 评论组件
date: 2023-11-15 22:03:29
categories: 工具使用
tags:
    - blog
excerpt: 在 hexo 框架，Redefine 主题中启用 Gitalk 评论组件
---

## 前言

-   原本我使用的评论系统是 Waline，配置较为繁琐，体验颇为一般，但也还算过得去。前些日子突然发现挂了，莫名其妙啊。
-   既然如此，那我就换一个评论系统好了。
-   得知 Gitalk 无需服务器也无需托管后，本人决定使用该评论系统。

## 启用

-   由于 Redefine 主题内置了启用 Gitalk 所需的相关代码和组件，根据 Redefine 提供的教程便可完成安装。

{% notel orange fa-triangle-exclamation **注意** %}

1. 存放 Gitalk 评论内容的 GitHub 仓库不能为 Private
2. Callback URL 必须严格与博客网址相符（不能有 www，且末尾需添加 “/”）
   {% endnotel %}
