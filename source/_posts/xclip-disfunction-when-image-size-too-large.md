---
title: xclip 在图片文件大小过大时不工作
date: 2023-08-27 21:40:17
categories: 工具使用
tags:
    - maim
    - screenshot
    - xclip
excerpt: xclip 在复制过大的图片文件时不工作的解决方案
---

## 问题描述

-   在[先前的一篇博文](https://blog.imlast.top/2023/08/15/%E4%BD%BF%E7%94%A8-maim-%E6%9B%BF%E4%BB%A3-flameshot-%E4%BD%9C%E4%B8%BA%E6%88%AA%E5%9B%BE%E8%BD%AF%E4%BB%B6/)中，我使用 maim 代替了 flameshot 作为截图软件，并且设置了相应的快捷键：

```config
# screenshot to clipboard
ctrl + alt + a
    maim --select | xclip -selection clipboard -target image/png

# screenshot to file
ctrl + alt + s
    maim --select ~/Pictures/screenshot/$(date +%Y-%m-%d_%H-%M-%S_maim | tr A-Z a-z).png
```

-   然而，我发现每当我想要截取整个屏幕范围并使用 xclip 复制到他处时，不是无法粘贴就是直接卡死。
-   经查询得知，这是因为 xclip 没有对于图片文件的支持，导致其在复制图片时会直接将所有的图片数据保存在系统剪贴板中。当图片文件过大时就会引发问题。

## 解决方案

-   参照该 [issue](https://github.com/hluk/CopyQ/issues/1070#issuecomment-935646874) ，解决方案的思路是将图片文件保存在 `/tmp/img.png` ，xclip 仅仅复制图片文件的链接。

-   修改快捷键：

```config
# screenshot to clipboard
ctrl + alt + a
    maim -su -b 2 /tmp/img.png; \
    echo "file:///tmp/img.png" | xclip -selection clipboard -target text/uri-list
```

-   顺便加上 dunst 提示：

```config
# screenshot to clipboard
ctrl + alt + a
    maim -su -b 2 /tmp/img.png; \
    echo "file:///tmp/img.png" | xclip -selection clipboard -target text/uri-list; \
    notify-send -u low -t 3000 "Screenshot copied to clipboard"

# screenshot to file
ctrl + alt + s
    filename=`date +%Y-%m-%d_%H-%M-%S_maim | tr A-Z a-z`; \
    maim -su -b 2 ~/Pictures/screenshot/$filename.png; \
    notify-send -u normal -t 3000 -i ~/Pictures/screenshot/$filename.png "Screenshot taken"
```
