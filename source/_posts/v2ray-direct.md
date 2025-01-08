---
title: v2ray 特定 IP 直连
date: 2025-01-01 18:04
categories: 工具使用
tags:
    - proxy
    - v2raya
excerpt: 阻止 v2ray 代理通往特定 IP 的流量
---

## 前言

-   本人与服务器之间的 ssh 连接也被 v2ray 代理，导致延迟很高甚至偶发断连。遂尝试配置 v2ray 直连服务器 ip。

## 正文

-   翻阅资料后，本人首先尝试修改 v2ray 配置 `/etc/v2ray/config.json` 中的 `ruleObject`:

    ```json
    "routing": {
      "domainStrategy": "IPOnDemand",
      "rules": [
        {
          "type": "field",
          "ip": [
            "<server-ip>"
          ],
          "outboundTag": "direct"
        },
      ]
    },
    ```

-   重启 `v2ray` 后发现不起作用。

---

-   后于 [v2rayA 官方文档](https://v2raya.org/docs/advanced-application/intranet-direct/) 发现直连方式，经测试有效。
-   但该方法会导致命令行中使用 `git push` 时无法连接，原因未知。

## 总结

-   说实话，这个方法十分粗暴。我想应该是有更加优雅的配置方式的，只是我没什么时间精力研究了，能用就行吧。
