---
title: 在 Linux 上使用 VIA
date: 2024-08-09 11:26:47
categories: 工具使用
tags: keyboard
excerpt: 在 Linux 上浏览器中使用 via 对键盘进行改建
---

## 前言

-   今日想对机械键盘进行改键，需要使用 [via](usevia.app)。该网站在 Windows 10 上运行良好，但是在 Linux 上出现了无法连接键盘的问题。

## 查看浏览器日志

-   本人使用的是 Brave 浏览器，系 Chrome 的二次开发版本，故访问[此页面](brave://device-log/)查看设备连接相关的日志。
-   在其中可以看到这样一条：

    ```plaintext
    Failed to open '/dev/hidraw3': FILE_ERROR_ACCESS_DENIED
    Access denied opening device read-write, trying read-only.
    ```

-   查看文件权限：

    ```bash
    ls -l /dev/hidraw3
    ```

-   output:

    ```plaintext
    crw------- root root 0 B Fri Aug  9 11:06:55 2024  /dev/hidraw3
    ```

-   可以看到，浏览器没有权限访问该文件。

## 授予权限

-   使用 chmod：

    ```bash
    sudo chmod a+rw /dev/hidraw3
    ```

-   回到 via 界面，刷新后即可连接设备进行改建。

---

-   收回权限：

    ```bash
    sudo chmod 600 /dev/hidraw3
    ```

-   直接重启系统也可以重置该文件的权限。
