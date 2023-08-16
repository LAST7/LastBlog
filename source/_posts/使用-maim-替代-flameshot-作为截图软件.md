---
title: 使用 maim 替代 flameshot 作为截图软件
date: 2023-08-15 19:58:09
categories: 记录
tags:
    - 截图
    - wm
    - 配置
excerpt: 记录使用 maim 替代 flameshot 的过程
---

## 前言

-   在刚刚装上 archlinux 的时候本人使用的是 Plasma KDE，截图工具是 flameshot。这是一款相当好用的截图软件，功能上基本与 Windows 上的 [snipaste](https://www.snipaste.com/) 相当。
-   然而在本人更换 de 至 bspwm 之后，flameshot出现了一些问题：在双屏（笔记本外接显示器）的情况下，flameshot 运行正常。拔掉外接显示器后，flameshot 可以启动，但是无法选取截图范围，并且无法看到其界面。
-   对本人来说无法选取截图范围是绝对不能接受的，必须寻找替代品。

---

-   需求很简单：轻量，可以选区，可以复制到剪贴板以及保存至本地。
-   尝试过的选择如下：
    -   **scrot**：截图时选取边框会消失一部分并不断闪烁，使用 `-f` flag 时若激活窗口为 nvim 则会很快退出截图
    -   **deepin-screenshot**：无法复制到剪贴板（xclip），且每次保存都需要手动选择路径
    -   **spectacle**：plasma 大礼包 口区
    -   **shutter**：就一截图软件你还有个客户端？
-   最终尝试 maim 之后感觉基本完美，符合需求。

## 安装

-   使用 pacman：

```bash
sudo pacman -S maim
```

## 设置快捷键

-   在 `sxhkdrc` 文件中加入如下配置：

```config
# screenshot to clipboard
ctrl + alt + a
    maim --select | xclip -selection clipboard -target image/png

# screenshot to file
ctrl + alt + s
    maim --select ~/Pictures/screenshot/$(date +%Y-%m-%d_%H-%M-%S_maim | tr A-Z a-z).png
```

{% notel blue fa-circle-exclamation **注意** %}
需要提前创建 `~/Pictures/screenshot` 文件夹。
{% endnotel %}

## Picom 相关问题

-   参考 [GitHub](https://github.com/naelstrof/maim/issues/172)，在 picom 的背景模糊作用下，maim 的选取窗口也会受到影响，需要在 `picom.conf` 文件中添加如下配置：

```config
blur-background-exclude = [
    # ...
    "class_g     = 'slop'",
    # ...
];
```

-   关闭选区窗口的动画效果：

```config
animation-exclude = [
    # ...
    "class_g = 'slop'",
    # ...
];
```

---

-   该 issue 内还有人提到 shadow 也会影响到截图的选取，可以以同样的方式将其排除。
-   由于本人的 picom 未开启 shadow，故不作记录。
