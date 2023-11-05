---
title: 彻底删除 Ubuntu 22.02 中的 Snap
date: 2023-11-05 21:20:57
categories: 记录
tags:
    - server
excerpt: 彻底删除 snap
---

## 前言

-   由于 Snap 软件包体积过大，后台进程会导致系统卡顿等等原因，最好删除 Snap。

## 删除 Snap

已经确认 snapd 是无法禁用的，只能强制删除。

1. 删除所有已安装的 Snap 软件

```bash
for p in $(snap list | awk '{print $1}'); do
    sudo snap remove $p
done
```

一般需要执行 2-3 次，直到出现如下提示：

```bash
No snaps are installed yet. Try 'snap install hello-world'.
```

2. 删除 Snap 的 Core 文件

```bash
sudo systemctl stop snapd
sudo systemctl disable --now snapd.socket

for m in /snap/core/*; do
    sudo umount $m
done
```

3. 删除 Snap 的管理工具

```bash
sudo apt autoremove --purge snapd
```

4. 删除 Snap 的目录

```bash
rm -rf ~/snap
sudo rm -rf /snap
sudo rm -rf /var/snap
sudo rm -rf /var/lib/snapd
sudo rm -rf /var/cache/snapd
```

5. 配置 apt 参数：禁止 apt 安装 snapd

```bash
sudo sh -c "cat > /etc/apt/preferences.d/no-snapd.pref" << EOL
Package: snapd
Pin: release a=*
Pin-Priority: -10
EOL
```
