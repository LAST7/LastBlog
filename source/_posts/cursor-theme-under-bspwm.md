---
title: 在 Bspwm 下设置鼠标主题
date: 2024-06-02 23:54:13
categories: 记录
tags:
  - bspwm
  - linux
excerpt: 在 bspwm 窗口管理器下设置统一的鼠标主题
---

## 前言

- 最近换了新电脑，系统迁移的其他部分都很顺利，唯独鼠标主题始终无法成功设置。
- 本人此前使用的是 LxAppearance，讲真，就算是在此前的那一台电脑上这个设置的过程也是十分曲折的，曲折到我自己都没明白这其中到底是哪个步骤起了作用。那时候也没有写博客的习惯，于是便留下了未来的自己在此发愁。
- 网上的绝大部分教程，包括 Arch Wiki 中详尽的指导，都没能完美设置 cursor theme。其中进展最多的仍然是 LxAppearance 的方案，然而且不提鼠标移动到其他应用窗口上之后就“原形毕露”，重启系统/桌面环境后鼠标就完全回到了系统默认的 Adwaita。

## 正文

- 好在经过孜孜不倦的翻阅互联网，本人终于在 Gist 上找到了一篇完美对症的教程：[Cursor theme.md](https://gist.github.com/abairo/5e2ed2faeadcc7abf43cda37826b2fbc)。
- 虽然按照其中的指示就可以完美设置鼠标主题了，不过本人在实际操作中有几个步骤存在改动，故留此博文以作记录。

### 安装主题

- 使用 pacman：

```bash
sudo pacman -S bibata-cursor-theme
```

- 该命令运行完成后，鼠标主题将被安装到 `/usr/share/icons/`

### 链接主题

- 与教程中不同，本人采取了软链接的方式将鼠标主题文件链接至 `~/.icons`

```bash
ln -s /usr/share/icons/Bibata-Modern-Classic ~/.icons/
```

### 配置系统鼠标主题

```bash
echo 'Xcursor.theme: Bibata-Modern-Classic' >> ~/.Xresources
xrdb ~/.Xresources
```

- 重启系统/桌面环境后生效。
