---
title: 在 Netlify 上部署 React 网站
date: 2023-11-12 15:08:17
categories: 工具使用
tags:
    - netlify
    - react
excerpt: FullStackOpen 中部署应用站点 Fly/Render 的一种替代方案
---

## 前言

-   做到 FullStackOpen Part3，遇到了需要部署在网络上的 React App，由于课程提供的 Fly 需要填写银行卡信息，以及 Render 在课程的 Part 11 可能出现的问题，我决定使用之前部署博客的 Netlify 来进行尝试。

## 正文

### 安装 netlify-cli

-   使用 npm:

```bash
npm install netlify-cli -g
```

-   其中 `-g` 参数表示全局安装

{% notel red fa-triangle-exclamation **注意：** %}
在 Archlinux 上使用 npm 时，为了不污染系统环境，需要提前设置 npm 的 prefix 来更改安装位置。详见：[Archwiki](https://wiki.archlinux.org/title/Node.js#Allow_user-wide_installations)
{% endnotel %}

### Build Site

-   FullStackOpen 的项目是使用 vite template 创建的，`package.json` 已经写好 build 指令：`vite build`

```bash
npm run build
```

### Deploy Draft Site

-   这一步会将网站部署到一个“草稿网页”上，具体就是在指定的 url 前填上一些随机字符（？），用作网页预览。
-   使用 Netlify:

```bash
netlify deploy
```

-   第一次使用时会要求 authorization，在浏览器中完成相应操作即可。
-   部署位置需要选择 `dist`，否则网页无法正常运行，参考本文末尾。

### Deploy Site

-   在确认一切无误后，使用 Netlify:

```bash
netlify deploy --prod
```

## 坎坷

### 无法安装 netlify-cli

-   在使用 npm 安装 netlify-cli 时，npm 报错包含如下内容：

```error
...
npm ERR: cannot find module 'inherits'
...
```

-   于是我使用 `npm install inherits -g` 手动安装了 inherits 包，但报错还是依旧。

---

-   再次仔细查阅报错后发现以下内容：

```error
...
npm ERR! sharp: Detected globally-installed libvips v8.14.5
...
```

-   因为有过几次折腾 LinuxQQ 闪退问题的经验，我知道这个包是 LinuxQQ 所需要的，使用 `pacman -Qi libvips` 查看后发现该包 Only _Required By: linuxqq_
-   抱着试一试的心态，我直接卸载了 LinuxQQ 及其依赖（包括 libvips），再次执行 `npm install netlify-cli -g` 后顺利完成安装。

---

-   现在回想一下可能是我的 LinuxQQ(libvips) 并非最新版本导致的，然而已经无从求证了。

### 部署目录错误

-   如果在 Deploy 时没有进行 Build 操作，或是选错了部署目录，则会在网页终端中看到如下报错：

```error
Expected a JavaScript module script but the server responded with a MIME type of "application/octet-stream"...
```

-   需要在执行 `netlify deploy` 或 `netlify deploy --prod` 的时候，将部署目录指定为 `dist`。
