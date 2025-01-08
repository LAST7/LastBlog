---
title: 使用 Let's Encrypt 为服务器配置 SSL Certificate
date: 2023-11-25 20:59:38
categories: 工具使用
tags:
    - certbot
    - nginx
    - server
    - cloudflare
excerpt: 使用免费的 Let's Encrypt 为 Nginx 服务器配置 SSL Certificate 来保证 https 连接
---

## 前言

-   部署在 Netlify 上的 React 前端发来悲报：既然客户端与前端服务器的连接是 HTTPS，那么前后端之间的连接也必须是 HTTPS，无法使用 ip 直连服务器了。
-   需要配置 ssl 认证才能启用 https 连接，鉴于企业级认证的费用过于昂贵，笔者选择使用 let's encrypt 提供的免费 ssl 认证。

## Let's Encrypt

-   Let's Encrypt 是一个由非营利性组织互联网安全研究小组（ISRG）提供的免费、自动化和开放的证书颁发机构（CA）。
-   本文使用 Certbot 完成 SSL 认证的获取，详见其[官方文档](https://certbot.eff.org/)，或是[这篇教程](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04)。

### 安装 Certbot

-   使用 apt：

```bash
sudo apt install certbot python3-certbot-nginx
```

### 获取 SSL 认证

-   使用 Certbot：

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

根据提示输入邮箱并同意条款。

-   成功后，会出现以下提示：

```plaintext
OutputIMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/example.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/example.com/privkey.pem
   Your cert will expire on 20xx-xx-xx. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot again
   with the "certonly" option. To non-interactively renew *all* of
   your certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:
   ...
```

留意证书文件和密钥文件的保存位置：`/etc/letsencrypt/live/example.com/`

## Nginx

-   在 Nginx 中启用 ssl，在主配置文件中 http 中添加如下配置：

{% notel orange fa-triangle-exclamation **注意** %}
下方配置中，有许多笔者未能完全理解，请谨慎参考。
{% endnotel %}

```conf
ssl_protocols TLSv1.2 TLSv1.3;

server {
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
}
```

## Cloudflare

-   本人的域名使用 Cloudflare 的 DNS 服务，可以在域名相关的配置选项中找到 “SSL/TLS" > "Overview"，将 "SSL/TLS encryption mode" 更改为 **"Full"**

## 测试

-   可使用 [Qualys SSL Labs](https://www.ssllabs.com/index.html) 对域名进行测试。
-   经过上述流程配置后的域名连接 SSL 评级应为 "A" 级。
