---
title: 解除移动端 30 系列显卡功率限制
date: 2024-02-20 20:51:59
categories: 记录
tags:
    - linux
    - nvidia
    - laptop
excerpt: 在 Linux 系统上解除 nvidia 3060-laptop 显卡的最大功率限制
---

## 前言

-   几个月前研究 Linux 系统上 Nvidia 显卡驱动的时候就发现，本人的 3060-laptop 最大功率被限定在了 115W，无法达到硬件最高的 130W 限制。然而在 Windows 系统上却没有出现这个问题。
-   发现问题后本人前去搜索引擎和各大论坛查找相关情况和解决方案，看到有人称其为“新版本驱动的 bug，目前无法解决”后便选择放弃了。

## 解决方案

-   然而近日一位 id 为“Icebooy”的群友在 [Nvidia 官方论坛](https://forums.developer.nvidia.com/t/power-limit-on-3000-mobile-series/193443/23)上发现了对应的解决方案，需要启用 `nvidia-powerd` 服务。

### nvidia-powerd

-   启用 `nvidia-powerd` 服务之前：

![](https://s2.loli.net/2024/02/20/HJsO8P1lGfcRBZ9.png)

-   可以看到最大功率被限制在 115W

-   使用 systemd 启用并设定开机自启：

```bash
sudo systemctl enable --now nvidia-powerd.service
```

-   启用 `nvidia-powerd` 服务之后：

![](https://s2.loli.net/2024/02/20/fKYD8ChAetXLEUv.png)

-   完美解决。

## 引用

-   [power-limit-on-3000-mobile-series](https://forums.developer.nvidia.com/t/power-limit-on-3000-mobile-series/193443/23)

> You will need to run nvidia-powerdd service which will increase the power drawn when you will run GPU extensive applications or benchmarks.
> ...

-   Arch Linux 交流群：583376491
