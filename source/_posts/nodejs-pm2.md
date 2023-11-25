---
title: 使用 pm2 进程化 Node.js 后台应用
date: 2023-11-18 20:54:08
categories: 工具使用
tags:
    - Nodejs
    - pm2
    - server
excerpt: 使用 pm2 对 Node.js 后台应用进程化
---

## 前言

-   FullStackOpen 的课程内容来到了后端应用编写的阶段，作业要求将后端进程部署到网络上，可以使用免费的后端部署平台 (Fly, Render)。
-   本人打算使用个人服务器来完成部署，然而在服务器上使用命令行启动的应用随着 ssh 连接的关闭会一同终止，所以需要将其进程化，一直在后台运行。

## pm2

### 安装

-   使用 npm:

```bash
npm install pm2 -g
```

-   关于全局安装前缀位置的修改，详见这篇 [ArchWiki](https://wiki.archlinux.org/title/Node.js#Allow_user-wide_installations)。

### 使用

-   启动：

```bash
pm2 start index.js --name <name>
```

-   查看所有受 pm2 管理的进程运行状况：

```bash
pm2 ls
```

-   通过 web interface 监视 pm2 进程运行情况：

```bash
pm2 monitor
```

根据命令行提示完成授权后，可使用相同的账号在 https://app.ap2.io/ 查看对应的 Bucket。

-   查看 pm2 管理的进程的标准输出：

```bash
pm2 log
```

-   停止 / 重启 / 删除进程：

```bash
pm2 stop <proc>
pm2 restart <proc>
pm2 delete <proc>
```
