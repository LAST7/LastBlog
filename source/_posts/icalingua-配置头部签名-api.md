---
title: icalingua++ 配置头部签名 api
date: 2023-08-19 14:22:12
categories: 工具使用
tags:
  - icalingua
  - docker
excerpt: 为 icalingua++ 配置头部签名 api
---

{% notel orange fa-triangle-exclamation **注意** %}
由于 qsign 作者被抓，项目已被删除。
本文内容仅作留档，已无任何实际作用。
{% endnotel %}

## 前言

-   虽然在 icalingua 的 tg 群中早就听闻了 qq 强制要求头部签名 api 才能登录/发送消息的情况，然而我这数月之前登录上的 icalingua 一直到目前为止都能够正常登录和收发消息，且没有配置任何头部签名 api。
-   然而最近一次更新之后，我发现我的 icalinuga 也无法登录了。
-   由于本人非常喜欢 icalingua 的 ui 以及一些小功能（例如防撤回），不愿意使用 tx 的 linuxqq。无奈之下只得去折腾一下这个头部签名 api。
-   由于使用公共 api 有被封号的风险，所以本人还是选择自己本地部署 api。

## 使用 Docker 部署

-   根据 unidbg-fetch-qsign 的 [wiki](https://github.com/fuqiuluo/unidbg-fetch-qsign/wiki/%E9%83%A8%E7%BD%B2%E5%9C%A8Docker)，使用 docker 镜像部署的步骤非常简单，只需要执行下面这条命令：

```bash
docker run -d --restart=always --pull=always --name=qsign -p 8091:8080 ghcr.io/fuqiuluo/unidbg-fetch-qsign:master
```

{% notel blue fa-circle-exclamation **注意** %}
使用 `-p 8091:8080` 将主机的 8091 端口映射到容器内的 8080 端口
{% endnotel %}

-   此后需要在 icalingua 的登录界面填写 api key 的地址：`http://127.0.0.1:8091`，选择一个相对高版本的 qq 即可进行登录。

## 设备选择与同时在线问题

-   在选择使用 android qq 登录后，本人发现 iPhone 手机上相同帐号的 qq 被强制下线了。待到我手机 qq 重新登录后，icalingua 又被强制下线了。
-   经过一点摸索，发现选择 android pad qq 登录不会导致手机上的 qq 被强制下线。

---

-   tx 真恶心。

## 关于签名服务部署至云端的考虑

-   目前来说我是将该提供签名的服务部署在本机的 docker 上，开机时 docker 自启动，可以达到隐形运行的状态。
-   不过既然是在登录时填写 api key 的地址，那么应该可以将该服务部署到服务器上，减少本机的性能开销。
-   不过该服务基本没有什么开销，最大的开销可能还是在于 docker 服务本身，而部署在服务器上又会大大增加安全方面的隐患，感觉有些得不偿失，遂放弃。
