---
title: v2raya 开启 ipv6 支持
date: 2023-08-18 17:43:49
categories: 工具使用
tags:
    - proxy
    - v2raya
    - ipv6
    - claude
excerpt: 如何开启 v2raya 的 IPv6 支持
---

## 前言

-   近日打开 claude 的网页版的时候，发现“所在地区不受支持”。然而代理是开启的状态，并且也确实是连接至美国的节点，怎么会是呢？
-   一番研究后发现，claude 这个网站优先走的是 ipv6，而我的 v2raya 没有开启 ipv6 的支持，导致从 ipv6 检索到的 ip 所在地其实是在北京，从而无法使用网页版 claude。

## 开启 v2raya 的 IPv6 支持

-   在官方文档搜索后，发现仅有一条词条是 IPv6 相关的，且并不能提供什么帮助。
-   随后去 GitHub 检索后，在一个 [issue](https://github.com/v2rayA/v2rayA/issues/325) 中找到了解决方案：在 v2raya 启动时添加一个额外的 flag：`--ipv6-support=on`

---

-   根据该 issue 下的回帖，并不建议直接修改 `/usr/lib/systemd/` 下的参数，因为在此处修改的参数会在 v2raya 更新后被覆盖掉。推荐的方法是在 `/etc/systemd/` 下新建一个 `v2raya.service` 文件来覆盖原本的配置文件。
-   操作步骤：

1.

```bash
sudo systemctl edit v2raya.service
```

2.

在 `### Lines below this comment will be discarded` 的 **上方** 添加如下配置：

```config
[Service]
ExecStart=
ExecStart=/usr/bin/v2raya --log-disable-timestamp --ipv6-support=on
```

3.

重启 v2raya 服务：

```bash
sudo systemctl restart v2raya
```

-   访问 claude.ai，一切正常。
