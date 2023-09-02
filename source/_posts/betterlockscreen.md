---
title: betterlockscreen 锁屏配置
date: 2023-09-01 15:50:01
categories: 工具使用
tags:
    - wm
    - betterlockscreen
    - xautolock
    - dunst
    - shellscript
excerpt: 使用 betterlockscreen 和 xautolock 配置 bspwm 锁屏
---

## table of content

-   usage of betterlockscreen
-   usage of xautolock
-   usage of dunst
-   bug of betterlockscreen causing it inability to execute pre/post scripts
-   using `&` with the commands in shellscript to keep it running in the background

## 前言

-   说老实话，锁屏这东西并不是什么必需品。毕竟我电脑里并没有什么不能被人看到的东西，倘若完全不配置锁屏也几乎没什么影响。
-   然而，秉持着试一试以及对于折腾的渴望，我还是下载了这个在 bspwm 上可以使用的锁屏软件：betterlockscreen，结果便是一下子深陷泥沼，前前后后折腾了整整两天多。

## Betterlockscreen

-   Betterlockscreen 是一款快速，轻量化的 Linux 锁屏程序，支持设定锁屏壁纸以及各种效果（如模糊，像素化，降低亮度）。
-   其锁屏服务启动时，会自动暂停 Dunst 的消息推送，并且支持任意自定义的锁屏前/后脚本。

### 安装

-   使用 pacman：

```bash
sudo pacman -S betterlockscreen
```

### 使用

-   设定锁屏壁纸：

```bash
betterlockscreen -w ~/Pictures/wallpaper/LofiGirl.png
```

{% folding orange::LofiGirl.png %}
![LofiGirl](https://s2.loli.net/2023/09/02/sikFL5CzW1gMRIG.webp)

-   图片来自 wallpaper engine，搜索 Lofi Gril Authentic 即可找到。（有点像是 AI 生成的，不过好看就行）
-   本人使用 ffmpeg 截取了视频其中一帧：`ffmpeg -i input.mp4 -ss 00:00:01 -vframes 1 output.png`
-   太好看啦！

{% endfolding %}

-   锁屏：

```bash
betterlockscreen -l
```

-   ...

详见 `man betterlockscreen` 。

## Xautolock

-   xautolock 可以监控用户在 X 显示服务中的行为，如果在用户设定的时间长度内没有任何行为被检测到，则会执行任意用户定义的命令（如启动锁屏程序）。
-   该程序只负责在一段时间内没有检测到用户行为后启动用户指定的程序，而不做任何其他行为，所以锁屏的行为完全由用户指定的程序来执行。

### 安装

-   使用 pacman：

```bash
sudo pacman -S xautolock
```

### 使用

-   在十分钟内没有监控到用户行为的情况下，执行某命令：

```bash
xautolock -time 10 -locker some-command
```

-   在距离倒计时结束的 15 秒时，执行某命令（提醒）：

```bash
xautolock -time 10 -locker some-command -notify 15 -notifier another-command
```

## Dunst 相关

-   Dunst 是一个高度可自定义化和轻量化的消息通知服务。

### 使用

```bash
dunstify "title" "content" -t 3000 -r 99 -u normal
```

-   `-t 3000`：消息窗口停留的时间，单位是毫秒（ms）
-   `-r 99`：设置消息 ID，若已经有相同 ID 的消息窗口则会将其覆盖
-   `-u normal`：设置消息的通知级别，共有三种：`low, normal, critical`。在 `~/.config/dunst/dunstrc` 文件中有相关样式的配置。

---

-   作为 xautolocker 的提示服务：

```bash
xautolock -time 10 -locker some-command -notify 15 -notifier path/to/dunst/script
```

`dunst_script` :

```shell
#!/usr/bin/bash

start=$SECONDS
remaining_time=$(( 15 - (SECONDS - start) ))

# if keyboard/mouse input detected, stop warning
while [[ $remaining_time -gt 0 && $(xprintidle) -gt 2000 ]]; do
    dunstify "betterlockscreen" "locking in $remaining_time" -t 1100 -r 99 -u normal

    sleep 1
    remaining_time=$(( 15 - (SECONDS - start) ))
done

# close the last message to prevent it shows up after unlock
dunstify -C 99

# info the user if the timer is stop by keyboard/mouse input
if [[ $(xprintidle) -lt 2000 ]]; then
    dunstify "betterlockscreen" "lock timer refreshed" -t 3000 -r 99 -u normal
fi

```

-   该脚本会在即将锁屏的 15 秒前使用 dunst 来向用户展示一个 15 秒的倒计时。

## 最终配置

-   `idle`：写入了 bspwmrc，开机时自动运行。

```shell
#!/usr/bin/bash

dunstify "betterlockscreen" "lock timer started." -t 2000 -r 99 -u normal

# kill xautolock if it running
pgrep xautolock > /dev/null && kill $(pgrep xautolock)

# start the timer
xautolock -time 10 -locker "$HOME"/dotfile/script/betterlockscreen/lockscreen -notify 15 -notifier "$HOME"/dotfile/script/betterlockscreen/lockwarn &
```

{% notel orange fa-triangle-exclamation **注意** %}
在 shellscript 中执行某项需要在后台持续运行的程序时，需要在其末尾加上 `&`，否则在 shellscript 中所有代码运行完成后该程序会跟着一起退出。
{% endnotel %}

---

-   `~/dotfile/script/betterlockscreen/lockscreen`：实施锁屏。

```shell
#!/usr/bin/bash

# lock screen
betterlockscreen -l dim
```

{% note info %}
dim 的百分比写在 `~/.config/betterlockscreen/betterlockscreenrc` 中
{% endnote %}

---

-   `~/dotfile/script/betterlockscreen/lockwarn` ：在即将锁屏的 15 秒前警告用户。

```shell
#!/usr/bin/bash

start=$SECONDS
remaining_time=$(( 15 - (SECONDS - start) ))

# if keyboard/mouse input detected, stop warning
while [[ $remaining_time -gt 0 && $(xprintidle) -gt 2000 ]]; do
    dunstify "betterlockscreen" "locking in $remaining_time" -t 1100 -r 99 -u normal

    sleep 1
    remaining_time=$(( 15 - (SECONDS - start) ))
done

# close the last message to prevent it shows up after unlock
dunstify -C 99

# info the user if the timer is stop by keyboard/mouse input
if [[ $(xprintidle) -lt 2000 ]]; then
    dunstify "betterlockscreen" "lock timer refreshed" -t 3000 -r 99 -u normal
fi

```

## 坎坷

### Betterlockscreen

-   在配置 betterlockscreen 的过程中我遇到了一个 bug：锁屏后 Dunst 的消息并没有被暂停。
-   经过简单的测试发现，本应当在锁屏被解锁后执行的命令在锁屏开始后立刻被执行了，相当于 Dunst 的消息通知被暂停后立马又恢复了。
-   在 GitHub 上发现了一个相关的 [issue](https://github.com/betterlockscreen/betterlockscreen/issues/389)，与仓库作者一同测试多次后找到了 bug 所在：程序的某个 flag 被后续的一行代码覆盖。将覆盖的代码删除后该 bug 得到了解决。
-   此外，本人还发现，尽管该程序的配置文件已经从 `~/.config/betterlockscreenrc` 迁移到了 `~/.config/betterlockscreen/betterlockscreenrc` ，但是用户自定义的锁屏前/后脚本只有在 `~/.config` 目录下才会得到执行。查看源代码后发现是配置文件迁移时，执行用户自定义脚本的代码并没有被更新。将该问题反映给作者后，也得到了解决。

### 命令后台执行

-   起初的时候本人并不知道尾部不加 `&` 的命令会在脚本执行完毕后被一同退出，每次执行完 `idle` 脚本后就会发现 xautolock 并没有被启动，还以为是 xautolock 出现了什么 bug。
-   后来询问 Claude 后发现了这个问题。有趣的是，虽然 Claude 帮助我发现了这一问题，但是它给出的解决方案是添加 `-foreground` 这一并不存在的 flag，可以说完全是胡编乱造……在后续通过 bing 搜索后才得到了正确的解决方式。

### 自制脚本

-   在发现了 betterlockscreen 这一 bug 后，本人因为没有能力自己定位 bug 所在，于是只能等待作者的回复。在此期间，本人想方设法写了一个自制脚本，其中包含手动执行关闭 Dunst 通知，检测系统空转时间等等功能的代码。
-   然而在自制的脚本编写完成，并测试通过后，Betterlockscreen 的作者发现了 bug 所在，并予以了修复……
-   花了一下午功夫的脚本就这么白写了，嗯……
