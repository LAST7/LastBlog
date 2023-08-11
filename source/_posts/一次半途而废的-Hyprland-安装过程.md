---
title: 一次半途而废的 Hyprland 安装过程
date: 2023-08-10 20:15:23
categories: 记录
tags:
    - wayland
    - hyprland
    - wm
excerpt: 记录未完成的 Hyprland 安装/配置过程
thumbnail: https://s2.loli.net/2023/08/10/TzL1Jayn4D7tZRf.webp
---

## 前言

-   [Hyprland](https://hyprland.org/) 是一款基于 wlroots 的动态平铺 [Wayland](https://wiki.archlinux.org/title/Wayland) 合成器，美观与性能兼具。
-   对于 hyprland，在我使用 bspwm 之前就已经早有耳闻，在 b 站也看到过不少用户使用他打造的桌面。说实话，确实是漂亮地很呐。
-   最近由于一系列原因（picom 动画 bug，x11 不支持多屏不同缩放），以及被 hyprland 优秀的外观吸引，使我想要尝试一下这款漂亮的 wm。

---

-   如标题所示，这次安装/配置最终结果是半途而废了，不过我仍然打算写下这篇博客，以便将来 hyprland 更加完善，更加稳定的时候（抑或者更换设备时）再来参阅。

## 安装

### Nvidia-drm

-   本人使用的是 nvidia 显卡，该显卡品牌以 "so fxxk you, nvidia" 著称，并且对于 hyprland/wayland 的支持并不很好。
-   经查询得知，想要开启能够调用 nvidia 显卡的 hyprland，必须启用 nvidia-drm。

---

-   在 `/etc/mkinitcpio.conf` 中的 `MODULES` 中加入以下配置：

```conf
MODULES=(nvidia nvidia_modeset nvidia_uvm nvidia_drm)
```

-   保存后运行 `sudo mkinitcpio -P` 来使之生效。

-   在 `/etc/default/grub` 中的 `GRUB_CMDLINE_LINUX_DEFAULT` 的值中加上:

```conf
nvidia_drm.modeset=1
```

-   保存后运行 `sudo grub-mkconfig -o /boot/grub/grub.cfg` 来使之生效。

{% notel blue fa-circle-exclamation 注意 %}

这里之所以将该内核参数添加到 grub 配置中，是因为本人的电脑安装了双系统，是由 grub 进行启动的。

{% endnotel %}

{% folding green::ChatGPT 对于启用 nvidia-drm 原因的解释 %}

-   Enabling the `nvidia-drm` module is necessary in certain cases when using NVIDIA graphics cards with Wayland compositors like Hyprland. The `nvidia-drm` module provides direct rendering capabilities for the NVIDIA graphics driver in a Wayland environment. Here's why it might be necessary:

1. **Kernel Modesetting (KMS) for NVIDIA:** The `nvidia-drm` module allows for kernel modesetting (KMS) support specifically for NVIDIA GPUs. KMS ensures that the kernel takes control of the display hardware early in the boot process, before user-space components like the display server (Wayland compositor) are loaded. This helps in proper initialization of the display and prevents flickering or other display-related issues during boot.
2. **Wayland and Direct Rendering:** Wayland is a modern display protocol that aims to replace the X Window System. It provides direct rendering capabilities, which means that applications communicate directly with the display server and graphics hardware without the need for an intermediate layer like X.Org. Enabling the `nvidia-drm` module ensures that the NVIDIA driver can interact with Wayland directly, providing optimal performance and compatibility.
3. **Compositor and Hardware Acceleration:** Wayland compositors, like Hyprland based on wlroots, rely on hardware acceleration for smooth graphics rendering and user interactions. Enabling the `nvidia-drm` module allows the compositor to make full use of the NVIDIA GPU's capabilities, which results in better performance and responsiveness.
4. **Optimal Wayland Experience:** Enabling `nvidia-drm` ensures that your NVIDIA GPU works seamlessly with the Wayland compositor and provides a more integrated and efficient experience. Without it, you might experience compatibility issues, performance degradation, or even a fallback to software rendering, which would affect the overall quality of the user interface and graphics performance.

-   Keep in mind that the specifics of enabling `nvidia-drm` may vary based on your distribution and setup. Following the instructions provided by the compositor (Hyprland) and your distribution's documentation is crucial to ensure that you have the necessary components in place for a smooth Wayland experience with your NVIDIA graphics card.

{% endfolding %}

---

{% notel orange fa-triangle-exclamation **注意** %}

根据 [Archwiki](https://wiki.archlinux.org/title/NVIDIA#mkinitcpio)，
修改该配置后，每次系统内核/显卡驱动更新之后都需要重新执行 `sudo mkinitcpio -P`。

为了防止可能的遗忘，[Archwiki](https://wiki.archlinux.org/title/NVIDIA#pacman_hook) 还提供了一个 pacman hook。

---

{% folding gray::/etc/pacman.d/hooks/nvidia.hook %}

-   `/etc/pacman.d/hooks/nvidia.hook`：

```plaintext
[Trigger]
Operation=Install
Operation=Upgrade
Operation=Remove
Type=Package
Target=nvidia-open
Target=linux
# Change the linux part above and in the Exec line if a different kernel is used

[Action]
Description=Update NVIDIA module in initcpio
Depends=mkinitcpio
When=PostTransaction
NeedsTargets
Exec=/bin/sh -c 'while read -r trg; do case $trg in linux) exit 0; esac; done; /usr/bin/mkinitcpio -P'
```

-   务必保证 `Target` 项所设置的软件包与你在前面的安装过程中所使用的相符（例如`nvidia` 或 `nvidia-dkms` 或 `nvidia-lts` 或 `nvidia-ck-something`）。
-   由于本人使用的是 `nvidia-open` 开源驱动，所以第六行的 `Target` 的值为 `nvidia-open`。

-   **另**： `Exec` 那一行看起来非常复杂，是为了避免在 `nvidia` 和 `linux` 软件包都发生更新的时候重复运行 `mkinitcpio`。如果你觉得无所谓，可以删掉 `Target=linux` 以及 `NeedsTargets`，然后 `Exec` 就可以简化为 `Exec=/usr/bin/mkinitcpio -P`.

{% endfolding %}

{% folding green::ChatGPT 对于该步骤的解释 %}

-   The reason you need to run `mkinitcpio -P` after updating the kernel or the NVIDIA driver with these changes is to regenerate the initial ramdisk (initramfs) with the necessary modules and settings to ensure proper functionality during the boot process.

Here's a breakdown of what's happening:

1. **Kernel Modules:** When you add `nvidia`, `nvidia_modeset`, `nvidia_uvm`, and `nvidia_drm` to `MODULES` in `/etc/mkinitcpio.conf`, you're specifying that these NVIDIA-related kernel modules should be included in the initramfs. The initramfs is a small image loaded into memory during the boot process before the full operating system is loaded. It contains essential drivers and modules required to access and mount the root filesystem, among other things.
2. **Modesetting:** Setting `nvidia_drm.modeset=1` in the GRUB config enables kernel mode-setting (KMS) for the NVIDIA Direct Rendering Manager (DRM) driver. KMS is a feature that allows the kernel to take control of the display hardware early in the boot process, ensuring that the display is correctly initialized even before the full graphics driver (X server or Wayland) is loaded.
3. **Regenerating Initramfs:** When you update the kernel or the NVIDIA driver, the changes made in step 1 and step 2 might not automatically carry over to the new kernel or driver version. The initramfs contains a snapshot of the necessary modules and settings at the time it was generated. Therefore, you need to regenerate the initramfs to include the updated or new kernel modules and settings for the new environment.

-   By running `mkinitcpio -P`, you are essentially rebuilding the initramfs with the latest configuration and modules, ensuring that everything required for the new kernel or driver version is present during the boot process. This step helps prevent potential issues related to missing or outdated modules and settings.

-   Remember that proper configuration and maintenance are essential for a stable and functional system, especially when dealing with hardware-specific components like graphics drivers.

{% endfolding %}

{% endnotel %}

### 环境变量

-   在 nvidia 显卡上启用 Hyprland 需要额外的环境变量，根据 [Hyprland wiki](https://wiki.hyprland.org/Nvidia/) :

```conf
env = LIBVA_DRIVER_NAME,nvidia
env = XDG_SESSION_TYPE,wayland
env = GBM_BACKEND,nvidia-drm
env = __GLX_VENDOR_LIBRARY_NAME,nvidia
env = WLR_NO_HARDWARE_CURSORS,1
```

-   根据一篇微信公众号教程：

```conf
# qt 相关
env = QT_AUTO_SCREEN_SCALE_FACTOR,1
env = QT_QPA_PLATFORM,wayland;xcb
env = QT_WAYLAND_DISABLE_WINDOWDECORATION,1
env = QT_QPA_PLATFORMTHEME,qt5ct # 使用 qt5ct 修改 qt 主题
```

```conf
env = _JAVA_AWT_WM_NONEREPARENTING,1 # 修复可能出现的 java 程序问题
env = CLUTTER_BACKEND,wayland
env = GDK_BACKEND,wayland
```

## 配置

{% btn center large::待续::::fa-solid fa-fire %}
