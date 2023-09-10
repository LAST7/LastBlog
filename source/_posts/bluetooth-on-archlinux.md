---
title: Arch Linux 使用蓝牙耳机
date: 2023-09-10 22:19:43
categories: 工具使用
tags:
    - bluetooth
excerpt: 在 archlinux 上使用命令行工具连接蓝牙耳机
---

## 前言

-   此前使用 KDE 的时候蓝牙连接颇为无脑，使用桌面环境自带的库和小组件即可。现在 bspwm 没有那些自带的库和组件了，不过命令行工具使用起来也没什么难度就是了。

## 启用蓝牙 && 安装相关组件

### Systemd

-   需要启动 systemd 的蓝牙服务：

```bash
sudo systemctl start bluetooth
```

-   或者设置为开机启动：

```bash
sudo systemctl enable bluetooth --now
```

### 相关蓝牙组件

-   音频控件：

```bash
sudo pacman -S pulseaudio pulseaudio-bluetooth
```

-   蓝牙控件：

```bash
sudo pacman -S bluez bluez-utils
```

## 配对 && 连接蓝牙

-   使用 `bluez` ：

```bash
bluetoothctl
```

-   进入相关命令行环境后：
    -   `help` ：显示帮助
    -   `scan on` ：打开扫描工具，新发现的蓝牙设备会被输出到命令行
    -   `pair <MAC>` ：与相应 MAC 码的设备配对
    -   `devices [Paired]` ：显示已经配对的设备
    -   `connect <MAC>` ：连接相应 MAC 码的设备
    -   `exit` / `quit` ：退出

## 设置 Pulse Audio 音频输出

-   首先在 'Playback' 页面中，将相关应用程序的音频输出重定向至蓝牙设备。
-   若没有声音，在 'Output Devices' 中检查蓝牙设备是否被静音。_（Pulse audio 对于新设备似乎是默认静音的，wtf？）_
