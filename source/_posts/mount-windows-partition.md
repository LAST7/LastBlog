---
title: Linux 系统下挂载 Windows 文件系统
date: 2024-06-03 00:13:30
categories: 记录
tags:
    - linux
excerpt: 通过修改 fstab 在 Linux 系统中挂载 Windows NTFS 文件系统
---

## 前言

-   同一台电脑双系统，如若不能互相挂载文件系统的话实在是有些过于不便了。在 windows 上挂载 Linux 的文件系统（视文件系统的种类）解决方案众多。例如 [WinBtrfs](https://github.com/maharmstone/btrfs) 挂载 btrfs 文件系统。

## 若无法手动挂载

-   修改 `/usr/share/polkit-1/actions/org.freedesktop.UDisks2.policy` 中 `<action id="org.freedesktop.udisks2.filesystem-mount-system">` 下的 `defaults` 为：

```xml
<defaults>
    <allow_any>auth_admin</allow_any>
    <allow_inactive>auth_admin</allow_inactive>
    <allow_active>yes</allow_active>
</defaults>
```

-   安装 ntfs 文件系统驱动：

```bash
sudo pacman -S ntfs-3g fuse
```

## 配置挂载

### 获取目标分区 UUID

-   使用 lsblk：

```bash
lsblk -f
```

-   NTFS 文件系统分区的 UUID 通常类似于如下形式：

```
42BE8456BE8443FF
```

### 修改 fstab 文件

-   在 `/etc/fstab` 文件末尾加上如下配置（以 D 盘为例）：

```
# Windows file sytems
# /dev/nvme0n1p4 LABEL=D
UUID=42BE8xxxxxxxx3FF   /run/media/last/D    ntfs-3g defaults,noexec,noatime,users 0 0
```

{% notel orange fa-circle-exclamation 注意 %}
这里的挂载点设置为 `/run/media/last/D` 是因为 memo 文件管理器默认将系统外其他分区挂载在此处。
{% endnotel %}

---

-   重启后生效。
