---
title: Let's Encrypt 泛域名证书申请
date: 2024-01-16 15:08:40
categories: 工具使用
tags:
    - certbot
    - nginx
    - cloudflare
    - server
excerpt: 使用 cetbot 为顶级域名及其子域名申请证书
---

## 前言

-   在服务器上部署 Koishi 的时候，本人发现使用带有额外路径的 nginx 流量反代会导致 koishi 控制台网页无法获取其网页脚本及样式表，同时也无法连接服务器上的 ws server。
-   经过了一翻探索之后，选择使用添加子域名配合 nginx 流量反代的方式。

-   然而此前本人为域名配置的 ssl 认证并非泛域名认证，子域名无法使用，因此需要配置一份泛域名认证以便于子域名的使用。

---

-   注：本文不包括 Cloudflare DNS 解析的内容。

## Certbot

### 安装

-   使用 apt：

```bash
sudo apt install certbot python3-certbot-dns-cloudflare
```

### 获取 Cloudflare API key

-   详见 [Cloudflare Global API Key](https://developers.cloudflare.com/fundamentals/api/get-started/)
-   创建一个名为 "cloudflare.ini" 的文件，并在其中填写如下信息：

```
dns_cloudflare_email = <youremail@example.com>
dns_cloudflare_api_key = <yourapikey>
```

-   将该文件设置为仅 root 身份可读。

### 获取认证

-   使用 certbot：

```bash
sudo certbot certonly --dns-cloudflare \
--dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
-d example.com,*.example.com --preferred-challenges dns-01
```

-   有关 nginx ssl certificate 的配置详见 [这篇文章](https://blog.imlast.top/2023/11/25/ssl-certificate/)

## 总结

-   此前本人的 nginx 流量反代一直是使用 location 地址块完成的。这种方式由于只能写在同一个 server 块中所以无法做配置文件的分离。
-   而通过子域名的方式配置多用途域名的方式则没有这个问题，唯二的代价便是：

1. 需要配置泛域名认证
2. 每个子域名都需要在 Cloudflare 新增 dns 解析规则

-   优点则是：

1. 便于分离和管理 nginx 配置
2. 便于迁移
3. 不会出现重定向后无法拉取网页资源的情况

-   以后应当尽量使用子域名的方式，亦或是两者结合。
