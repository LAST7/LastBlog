---
title: kitty 简单配置
date: 2023-10-02 23:36:45
categories: 工具使用
tags:
    - kitty
    - terminal
    - zsh
excerpt: 从 alacritty 到 kitty
---

## 前言

-   alacritty 一切都好，满足了我的大部分需求，所以即便已然多次看到对于 kitty 的赞美我也没有为之所动。
-   不过自从我发现 nvim 中的 bufferline 插件的 underline indicator 无法正常显示的原因在于 alacritty 后，我就有些动摇了。尝试一下，又有何妨？

## 安装 kitty

-   使用 pacman:

```bash
sudo pacman -S kitty
```

## 配置 kitty

### Font

-   字体相关：

```conf
font_family      FiraCode Nerd Font
bold_font        auto
italic_font      auto
bold_italic_font auto
```

```conf
font_size 16.0
```

-   下划线：

```conf
modify_font underline_thickness 200%
modify_font underline_position  4
```

### Cursor

-   形状：

```conf
cursor_shape beam
```

-   闪烁频率（0 为不闪烁）：

```conf
cursor_blink_interval 0
```

### Terminal Bell

-   关闭音效：

```conf
enable_audio_bell no
bell_on_tab no
```

### Keyboard Shortcuts

-   复制粘贴：

```conf
map ctrl+shift+c  copy_to_clipboard
map ctrl+shift+v  paste_from_clipboard
```

### Window Mangement

-   Tab Management:

```conf
map alt+]             next_tab
map alt+[             previous_tab
map alt+t             new_tab
map ctrl+w            close_tab
```

### Color Scheme

-   Catppuccin:
-   使用 kitty 自带的 theme selector：

```bash
kitty +kitten themes
```

-   在其中选择即可。

## 坎坷

### 莫名的消息弹窗

-   在终端切换目录时，会有消息弹窗提示当前目录。
-   原因是 oh-my-posh 触发了 osc99 相关组件（具体原理不明）。
-   解决方式：在 theme 的配置文件中将相关变量设置为 false：

```json
"osc99": false,
```

参考 [issue](https://github.com/kovidgoyal/kitty/issues/5172)

### fcitx5 IME

-   kitty 在 x11 桌面环境中默认情况下无法使用输入法，需要设置环境变量：

```plaintext
GLFW_IM_MODULE=ibus
```

参考 [issue](https://github.com/kovidgoyal/kitty/issues/469#issuecomment-419406438)
