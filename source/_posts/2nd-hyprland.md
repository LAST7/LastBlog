---
title: Hyprland 二周目
date: 2024-12-19 02:21:09
categories: 工具使用
tags:
    - wayland
    - hyprland
    - nvidia
excerpt: 记录再次尝试 hyprland 的过程，以及初步的配置结果
thumbnail: https://s2.loli.net/2023/08/10/TzL1Jayn4D7tZRf.webp
---

## 前言

### 初尝 Hyprland

-   大约半年前，我被 hyprland 各种绚丽的动效吸引，不顾 N 卡的艰难险阻在笔记本上安装了 hyprland。事实证明，这不是一次良好的体验。
-   回看当初的记录，其实大多数的麻烦都是 N 卡和输入法造成的。诸如 nvidia drm 和驱动钩子的设置，以及众多基于 Electron(Chromium) 的应用中，n 卡驱动造成的闪烁和输入法问题着实是费了我半天工夫。
-   而如今，我已改用 A 卡，顿感浑身清爽。

### 开箱即用 || 从零配置

-   在初入 DE Rice 领域的时候，我曾坚信只有由自己亲手去写（缝） dotfiles，搞明白每一个配置选项和桌面环境的启动流程，才能保证在某个桌面组件出问题的时候可以快速定位到问题源头并予以修复。直接套用他人提供的配置文件看似在配置阶段省下了不少时间，实则会在将来的某一刻加倍偿还。
-   虽然如今我依然这样觉得，但是已然不愿意付出折腾系统美化所消耗的时间和精力代价了。

---

-   所以这次我选择了一种折中的方案：套用 mylinuxforwork 的 [dotfile](https://github.com/mylinuxforwork/dotfiles)（该配置套件甚至在 Hyprland 的官方 [wiki](https://wiki.hyprland.org) 中得到推荐），在此基础之上加入/修改一些自己定制的配置方案。
-   当然，这一切也是基于我过去很长一段时间内使用 Bspwm 和上一次配置 Hyprland 的经验才能实现。毕竟，深度定制桌面环境配置的前提依然是能看懂各个配置项的作用。

### Nvidia

-   Nvidia 是令许多 Wayland 用户感到头痛的硬件，其驱动（厂商）与 Wayland 社区之间长久的矛盾导致 Wayland 对于 Nvidia（又或者说是 Nvidia 对于 Wayland）的支持在很长时间内都处于一个十分尴尬的情况。

-   根据这个 [Reddit Thread](https://www.reddit.com/r/linux/comments/1b61kiz/eli5_why_doesnt_nvidia_play_well_with_wayland/)，Wayland 与 Nvidia 之间的主要矛盾在于所使用的 GPU 缓存管理 API：Wayland 希望使用 GBM，而 Nvidia 则拒绝了 Wayland 的提案，要求使用自家开发的 EGLStream。
-   事实上，绝大部分的 Linux 桌面和显卡驱动生态中所使用的都是 GBM，而 EGLStream 则只有 Nvidia 在他们的驱动中使用。EGLStream 高度封装且闭源，倘若要将 EGLStream 推广到各个系统上，可能会导致其他的硬件厂商（AMD，Intel）为了兼容自己的硬件设备而开发各种各样的 API 扩展，这对于 Wayland 的社区开发者而言是一个绝对不想看到的局面。

-   此外，在 Wayland 的早期开发/讨论会议中，Nvidia 受邀但没有参加。这也是为什么这场关于 GBM vs EGLStream 的纷争持续了如此之久的原因之一。

-   在多年的协商后，Nvidia 最终还是为其驱动程序添加了 GBM API 接口。尽管这些接口的使用仍然不尽如人意，也总要好过 GBLStream 时期。

---

-   如日中天的 Nvidia 对于开源世界的傲慢姿态再一次体现了资本的最终目标是增值，除此以外的一切事物他们并不关心。

## 安装

-   Mylinuxforwork 的 dotfiles 提供了安装脚本，非常方便。安装脚本还自带了许多安装选项，甚至自带备份旧配置文件的功能。
-   具体的备份位置在 `~/.ml4w-hyprland/backup`。

---

-   注意到 ml4w 自动安装了一些我不需要的包，例如 `breeze` 和 `firefox` （我使用 Brave），遂在安装完成后将它们删除。
-   我本来以为删除 breeze 可能会导致某些 icon 出现问题，后来发现其实没什么影响。

## 配置介绍

### Waybar && Workspace

-   Wayland 下的 bar 一般使用 [waybar](https://github.com/Alexays/Waybar)，ml4w 的自带的设置中可以调整 waybar 中的许多配置项，例如时间显示格式，workspace 数量以及是否展示某些应用图标等等。
-   一开始我以为要想修改默认的 workspace 数量需要在 waybar 的配置文件中改，后来发现并非如此。
-   ml4w 配置中虽然确实仅有 waybar 的配置文件（`~/.config/waybar/modules.json`）中可以找到 workspace 相关的配置，但想要设置 hyprland workspace 相关的配置项需要在 `~/.config/hypr/` 目录下新建 workspace 相关的配置文件。

### Settings App && Waypaper

-   ml4w 中自带一个设置应用，可以用于设置许多配置中的选项。在 ml4w 中，作者并不建议用户直接修改默认的配置文件。正确的方式是新建一份自己的配置文件，然后在 Settings App 中选择自定义的配置文件。

-   ml4w 还有一个可以选择壁纸的应用（waypaper），选择完成后还会自动修改 waybar 和 kitty 等应用的主体色。
-   说实话，默认的这些个壁纸都不是很符合我的口味，然而也不愿再花大精力去挑选壁纸了，日后再议吧。

## 配置修改

### 新窗口设置为右侧打开

-   其实刚刚开始使用这份配置的时候我就发现了这个在上次使用 hyprland 的时候也遇到过的问题。上次的时候因为 n 卡和输入法的各种 bug 导致我还没来得及排查这个问题就直接放弃跑路了，这次终于得以窥得这其中的缘由和解决方案。
-   我开始的时候在 wiki 里找了半天，搜索了诸如 `workspace`, `window-rules`, `position` 等等，甚至还研究了一下 `special workspace`。最后在一个 reddit 论坛上发现，该配置项另有他人：
-   想要使新窗口始终在右侧打开，需设置 layout 规则如下：

    ```plaintext
    dwindle {
        ...
        force_split = 2
        ...
    }
    ```

-   说实话，不是很明白什么人会允许窗口的打开位置随着随手一挥而不知道在什么地方等待指令的鼠标一样不知所踪。

### 多显示器 Workspace 设置

-   尝试过修改 waybar 中的 `persistent-workspaces`，不起作用。
-   后在 hyprland 配置中增加如下内容：

    ```plaintext
    workspace = 1, monitor:DP-1, persistent:true, default:true
    workspace = 2, monitor:DP-1, persistent:true, default:true
    workspace = 3, monitor:DP-1, persistent:true, default:true
    workspace = 4, monitor:DP-1, persistent:true, default:true
    workspace = 5, monitor:DP-1, persistent:true, default:true
    workspace = 6, monitor:DP-2, persistent:true, default:true
    workspace = 7, monitor:DP-2, persistent:true, default:true
    workspace = 8, monitor:DP-2, persistent:true, default:true
    workspace = 9, monitor:DP-2, persistent:true, default:true
    workspace = 10, monitor:DP-2, persistent:true, default:true
    ```

-   另，加入如下的 keybindings 配置：

    ```plaintext
    bind = $mainMod, bracketleft, workspace, m-1
    bind = $mainMod, bracketright, workspace, m+1

    bind = $mainMod CTRL, H, movetoworkspace, m-1
    bind = $mainMod CTRL, L, movetoworkspace, m+1
    ```

-   其中，`m-1` 中的 `m` 代表 `monitor`，这样设置是为了防止在左边屏幕上切换 workspace 时不慎切到右边的屏幕上。
-   然而 hyprland 中的 workspace 似乎并不是在启动时初始化的，而是在被切换到的时候才会被初始化。这导致每次启动 hyprland 的时候，快捷键 `super + [/]` 切换不到任何 workspace 上（因为那时候只有 workspace 1 和 6 被初始化），之后在用鼠标点击 waybar 中的各个 workspace 一次，将它们手动初始化后才能正常切换。
-   一个不怎么优雅的解决方案是在 hyprland 中加入如下的配置：

    ```plaintext
    exec = for i in {1..10}; do hyprctl dispatch workspace $i; done && hyprctl dispatch workspace 1
    ```

-   之所以不那么优雅，是因为 workspace 1 是在左侧屏幕上的，该命令执行完毕而右侧的屏幕也会停留在 workspace 1 上，然而显示的窗口其实是 workspace 10 中的。

---

-   虽然不能复现 bspwm 中那样两块屏幕分离并可以独自循环的 Workspace，不过也算是可堪一用了。

### 窗口透明度调整

-   默认配置中，inactive 的窗口透明度会被调整为 0.8。对于某些窗口来说，这并非良好的体验。
-   调整如下：

    ```plaintext
    windowrulev2 = opacity 1.0 override 1.0 override 1.0, class:(brave-browser)
    windowrulev2 = opacity 1.0 override 1.0 override 1.0, class:(QQ)
    windowrulev2 = opacity 1.0 override 1.0 override 1.0, class:(discord)
    ```

## BUG 及修复方式

### Electron IME

1.  为 Electron 应用开启 wayland 平台特性以及输入法支持:

    ```plaintext
    --ozone-platform-hint=auto
    --enable-wayland-ime
    ```

2.  使用 `flag.conf` 的方式

### Brave

1.  与 Electron 应用相同，Chromium 也需要特定的启动 flag 来适配 wayland。若不添加，Brave 的弹出菜单会出现一个“透明”的边框。

2.  需要添加 `--gtk-version=4` 并设置 `GTK_IM_MODULE=fcitx` 以使用 fcitx5。 或开启 `wayland-ime` ，但会导致奇怪的 bug ：地址栏内输入的内容会重复出现两次，且输入的内容背景色为亮黄色，有些许不美观。

### OBS

-   切换至 Wayland 后 OBS 原本使用的屏幕源都无法使用了。经过一翻搜索和折腾，以及绕了一大圈原路，我终于是成功让 OBS 能够添加屏幕源了。

-   安装了 `pipewire`, `wireplumber`, `xdg-desktop-portal-hyprland`，在互联网上检索了半天，测试命令不下十几条，最终都没能让 `xdg-desktop-portal-hyprland` 服务跑起来。
-   倒是有个邪道方法：直接 `dbus-run-session Hyprland`，然后再 `Ctrl-C` 终止，就可以正常启动 `xdg-desktop-portal-hyprland`。你要问我为什么，那我只能说我也不知道。。

---

-   最后发现，其实根本不需要 `xdg-desktop-portal-hyprland` 作为 `xdg-desktop-portal` 的后端服务，只要后者正常运行 OBS 就可以捕捉屏幕画面，只要重新添加一个 PipeWire 的 Screen 源就好了。
-   此外，重启 OBS 之后该 screen 源会要求重新选择屏幕，并且处于黑屏状态。当我尝试使用 Del 键将其删除时，它又奇迹般的活了过来，令人迷惑不已。

### VLC

-   切换至 Hyprland 后使用 VLC 播放视频只有音频没有视频。
-   在 Tools -> Preferences -> Video 中，将 Output 调为 `Automatic` 后恢复正常。
-   我是什么时候又是为什么要把他调成 VDPAU 的来着？

### Timeshift

-   Timeshift 在 Wayland 上无法启动。
-   解决方案：

    1.  将 `/usr/share/applications/timeshift-gtk.desktop` 复制到 `~/.local/share/applications/timeshift-gtk.desktop`，修改其中的 `Exec`:

        ```plaintext
        Exec = bash -c 'pkexec env $(env) timeshift-launcher'
        ```

    2.  使用 `sudo -E timeshift-gtk` 来启动 Timeshift。

## Tips

### 查看显示器属性

-   使用 `hyprctl monitors all` 来查看连接到系统的显示器的属性。

### 查看窗口属性

-   在设置 Window Rule 的时候，我们需要指定规则生效的窗口名称(title），或是类别（class）。
-   我们可以使用 `hyprctl clients` 来查看当前打开的窗口的属性。

### 退出 Hyprland

-   某些 Hyprland 的配置项并不会立即生效，需要重启 Hyprland。
-   使用 `hyprctl dispatch exit` 来退出到 sddm 界面，输入登录密码重启 Hyprland。

## 结束

-   就算过了半年，Hyprland 依然是一款非常需要折腾的 wm，即使我已经使用了所谓开箱即用的配件库。
-   其中的麻烦有 Wayland 导致的，也有 Hyprland 导致的。Wayland 确实比 X11 年轻不少，但是自从项目自 2008 年正式启动以来也已过了 16 载有余了，项目的推进速度和 bug 修复速度真是慢的让人不知该说什么好。
-   话虽如此，还是不得不感叹，相较于 N 卡，A 卡折腾 hyprland 真是相当轻松。

---

-   最后还是要说些好的，wayland/hyprland 的动画效果真是舒服。X11 上依靠 picom 的动效相比之下就显得廉价不少了。

## 参考资料/网站

### General

-   [mylinuxforwork/dotfiles - Github](https://github.com/mylinuxforwork/dotfiles)
-   [Hyprland Wiki](https://wiki.hyprland.org/)

### Electron(Chromium) Wayland Support

-   [Wayland - Arch Wiki](https://wiki.archlinux.org/title/Wayland)
-   [Wayland: Electron - Arch Wiki](https://wiki.archlinux.org/title/Wayland#Electron)
-   [Chromium - Arch Wiki](https://wiki.archlinux.org/title/Chromium#Native_Wayland_support)

### Fcitx5

-   [Using Fcitx 5 on Wayland](https://fcitx-im.org/wiki/Using_Fcitx_5_on_Wayland)
-   [Brave Browser on Wayland instead of XWayland](https://www.reddit.com/r/brave_browser/comments/o4x1rr/brave_browser_on_wayland_instead_of_xwayland/)

### Workspace

-   [Confused how workspaces function on multiple screens. - Reddit](https://www.reddit.com/r/hyprland/comments/12op694/confused_how_workspaces_function_on_multiple/)
-   [Issue with waybar, multiple monitors and default/persistent workspaces - Reddit](https://www.reddit.com/r/hyprland/comments/1du9fkw/comment/lbfwfaq/)

### Screen Sharing(OBS)

-   [Screen sharing - Hypr Wiki](https://wiki.hyprland.org/Useful-Utilities/Screen-Sharing/)
-   [Hyprland-dexktop-portal](https://wiki.hyprland.org/hyprland-wiki/pages/Useful-Utilities/Hyprland-desktop-portal/)
-   [Screen sharing on Hyprland(Arch Linux) - Github Gist](https://gist.github.com/brunoanc/2dea6ddf6974ba4e5d26c3139ffb7580#troubleshooting-obs)
-   [Screen Capture - Arch Wiki](https://wiki.archlinux.org/title/Screen_capture#Wayland)
-   [How to capture screen use obs just black screen now? - Reddit](https://www.reddit.com/r/hyprland/comments/182q2tf/how_to_capture_screen_use_obs_just_black_screen/)

### VLC

-   [VLC media player is not displaying video, but audio works - AskUbuntu](https://askubuntu.com/questions/668834/vlc-media-player-is-not-displaying-video-but-audio-works)

### Others

[ELI5: Why doesn't Nvidia play well with wayland? - Reddit](https://www.reddit.com/r/linux/comments/1b61kiz/eli5_why_doesnt_nvidia_play_well_with_wayland/)
