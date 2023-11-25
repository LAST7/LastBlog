---
title: Nginx 流量反代
date: 2023-11-25 21:25:16
categories: 工具使用
tags:
    - nginx
    - server
excerpt: 使用 Nginx 实现服务器流量反向代理
---

## Nginx

-   **Nginx**（发音同“engine X”）是异步框架的网页服务器，也可以用作反向代理、负载平衡器和 HTTP 缓存。该软件由俄罗斯程序员伊戈尔·赛索耶夫（Игорь Сысоев）开发并于2004年首次公开发布。
-   **Nginx** 是免费的开源软件，根据类 BSD 许可证的条款发布。

---

-   本人选择使用 Nginx 为服务器提供流量反代服务，将特定通往特定网址的流量导向特定的端口。
-   这样一来，服务器只需开启常见的网络端口（如 80，443 等），而无需根据进程监听的端口频繁的修改防火墙规则。

## 安装

-   使用 apt：

```bash
sudo apt install nginx
```

### 配置

-   Nginx 的配置文件位于 `/etc/nginx/nginx.conf`

{% notel orange fa-triangle-exclamation **注意** %}
下方配置中，有许多笔者未能完全理解，请谨慎参考。
{% endnotel %}

-   在 _server_ 中添加如下配置：

```conf
location /phonebook {
    rewrite /phonebook/(.*) /$1 break;
    proxy_pass http://localhost:3001;
    proxy_set_header Host $host;
    proxy_buffering on;
    proxy_cache STATIC;
    proxy_cache_valid 200 30s;
    proxy_cache_use_stale error timeout invalid_header updating
                          http_500 http_502 http_503 http_504;
}

location /bloglist {
    ...
}
```

-   该配置会将指向 `<server-url>/phonebook/...` 的流量转往 `localhost:3001` ，且会包含 "phonebook" 之后的网址内容。
-   对于上文所示的各种配置，由于目前的主要目标是 FullStackOpen 的作业，笔者只是浅尝辄止，未求甚解。

## 启动

-   启动 Nginx：

```bash
sudo systemctl start nginx
```

-   修改配置后需重新加载 Nginx：

```bash
sudo nginx -s reload
```
