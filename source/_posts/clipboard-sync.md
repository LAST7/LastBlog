---
title: 关于 X11 与 Wayland 的剪贴板同步问题
date: 2026-02-14 02:56:36
categories: 工具使用
tags:
    - linuxqq
    - wayland
    - shell
    - python
excerpt: 关于 linuxqq 的剪贴板操作行为，以及 Linux 下 X11/Wayland 的剪贴板同步行为
---

## 前言

- 前些时日笔者就注意到 linuxqq 中复制内容实际上输出到了 X11 剪贴板中而非 Wayland，尽管 linuxqq 窗口确实是在 Wayland 混成器下运行的。推测可能是一些旧版本的 Electron 接口未能及时更新所致。

- 那时候，笔者所在群聊的群主用 AI 写了一个同步脚本，来将 x11 剪贴板中的内容同步到 Wayland 粘贴板。详见 [这篇文章](https://blog.imlast.top/2025/10/15/linuxqq-clipboard-issue/)。

## 两个悬而未决的问题：锁丢失和内容截断

- 在使用过程中，笔者发现该脚本出现了以下两个问题：
    1. 休眠（Hibernate）后，锁文件丢失导致多个脚本实例同时运行
    2. 体积较大的图片文件在会被“截断”，具体来说，粘贴出的图片数据不完整，通常表现为只有截图区域的上半部分

---

- 关于第一个问题，我一开始 **推测** 是放置锁的位置不对：`/tmp/clipboard_sync.lock`。原先猜想会不会是 Hibernate 清空了 `/tmp` 路径下的文件，但写了一个简单的脚本测试之后，发现并不会发生这种事情。后续我修改了脚本，将锁文件存放在 `$XDG_RUNTIME_DIR/` 路径下，至于是否会发生同样的问题，尚未进行充分的测试。

- 于是这个问题就变成了一个未解之谜……

- 就第二个问题来说，**推测** 是 shell 语言中的管道操作异步执行导致的，关键在于脚本的这一部分：

    ```bash
    # Pipe the image data directly to the Wayland clipboard
    xclip -selection clipboard -t "$x11_img_type" -o 2>/dev/null | wl-copy -t "$x11_img_type"

    # ...

    # Clear the X11 clipboard to make Wayland the single source of truth and prevent echo-syncing
    xclip -selection clipboard -i /dev/null
    ```

- 可以看到，在通过管道命令输出 Wayland 剪贴板的内容到 X11 剪贴板后，脚本几乎是立刻（中间还有一些设置变量的步骤）清空了 X11 剪贴板。如果此时管道传输没有结束，就会导致同步数据被截断的情况。

- 然后事后不论是直接测试这段逻辑，当时反复运行脚本，甚至手动删除锁文件运行多个脚本实例，都没有观察到类似的 bug 再次出现。

- 所以这也成了一个未解之谜……

## 重复同步

- 在我尝试使用 Python 编写一个相同逻辑的脚本之后，观察 log 发现：

    ```plaintext
    [15:58:10] ✓ WL -> X11 | Text:
    xclip 同步到 wl-clipboard 之后 xclip 被清空了，但是 wl-paste -...
    [15:58:11] ✓ X11 -> WL | Text:
    xclip 同步到 wl-clipboard 之后 xclip 被清空了，但是 wl-paste -...
    [15:58:19] ✓ WL -> X11 | Text:
    我之前看到同步两次的情况加了个锁避免写回，结果发现 x11 被清空了，然后又触发同步，wl-clip...
    [15:58:20] ✓ X11 -> WL | Text:
    我之前看到同步两次的情况加了个锁避免写回，结果发现 x11 被清空了，然后又触发同步，wl-clip...
    [15:58:21] ✓ WL -> X11 | Text:
    非常搞笑
    [15:58:22] ✓ X11 -> WL | Text:
    非常搞笑
    ```

- 可以看到，每次复制内容，脚本都执行了两次同步行为。脚本内监听 Wayland 的函数是这样的：

    ```python
    def watch_wayland():
        """Listener process: wl-paste --watch blocks the process"""

        # wl-paste will echo '1' whenever the clipboard changes
        proc = subprocess.Popen(
            ["wl-paste", "--watch", "echo", "1"], stdout=subprocess.PIPE
        )
        while True:
            if proc.stdout.readline():  # blocks until clipboard changes
                time.sleep(0.05)
                sync_wayland_to_x11()
    ```

- 不难想到，x11 -> wayland 的脚本同步行为也触发了 `wl-paste --watch echo 1` 的监听机制，从而使得已经同步到 Wayland 的内容被同步回了 X11。

- 于是我理所当然的想着加一个变量充当锁机制，在 x11 -> wayland 同步行为发生后让 `watch_wayland` 跳过下一次监听机制的触发：

    ```python
    def sync_wayland_to_x11():
        """Wayland (event driven) -> X11"""
        global last_text, last_img_hash, wl_ignore_next_update

        if wl_ignore_next_update:
            wl_ignore_next_update = False
            return

        # ...
    ```

    _(在运行 `wl-copy` 命令之前将 `wl_ignore_next_update` 设为 True)_

- 却发现从 linuxqq 中复制内容后剪贴板中什么都没有，何意味？
- 查看 log：

    ```plaintext
    [16:27:03] ✓ X11 -> WL | Text:
    xclip 同步到 wl-clipboard 之后 xclip 被清空了，但是 wl-paste -...
    [16:27:04] ✓ X11 -> WL | Text:

    ```

- 可以看到，日志最后的同步信息是一个 **空行**，意味着最后 Wayland 剪贴板被同步了成了“空”，导致粘贴行为什么都没有输出。

- 然后我使用 pdb 一步一步的执行脚本，并在旁边打开一个终端窗口时刻观察 X11 剪贴板中的内容变化，最终发现运行 `wl-copy` 同步 X11 剪贴板中的内容至 Wayland 剪贴板之后，X11 剪贴板中的内容被清空了。

- 这是为什么？

## 剪贴板所有权

- 原因在于系统剪贴板具有一个 `ownership` 的属性，这一点在 Wayland 和 XServer 上是一样的。

- 当脚本运行 `wl-copy` 来同步内容时，CLIPBOARD 这个 selection 的 ownership 就不归属于 linuxqq（准确的来说是 linuxqq 内部的某个陈旧的子模块）所有了。

- 事实上，笔者先前对于系统剪贴板的运作有一个误解——**误以为剪贴板是一个归属于 Wayland/XServer 所管理的 Buffer**，然而事实上在 Wayland/XServer 中复制粘贴这样的跨窗口数据通信，事实上是一种“拉取式”的行为，即当“粘贴”操作发生时，当前窗口会向剪贴板的 owner 发送一个请求获取 CLIPBOARD 中的内容。

- 这样一来，就不难理解为什么一旦运行 `wl-copy` 后，剪贴板中的内容就直接“消失不见了”——因为 ownership 已经不归属于 linuxqq，而是属于 Wayland Compositor。而同步脚本自身并没有 XWayland 那样同步剪贴板的行为，`xclip` 自然也无从获取数据。

---

- 对于这个问题，我能想到三种解决方法：
    1. 回到原先没有同步锁的逻辑， **任由第二次同步发生**（事实上是让该脚本始终掌握剪贴板的 ownership）
    2. 保留同步锁，但是“双写”，即每次同步都 **同时写入 Wayland 和 X11 的剪贴板**
    3. 跳过空内容的同步，即 **在 linuxqq 失去剪贴板所有权后，检测到 xclip 输出内容为空（有更新）则跳过同步**

- 对于 `1`，说实话这有点 “该系统依赖 bug 运行” 的味道，按照正常的逻辑 X11 同步到 Wayland 后不应该再触发一次 Wayland 同步回 X11 的行为，嗯，正常来说是这样的。

- 对于 `2`，这是最符合逻辑的做法，代价就是实现起来比较复杂，并且性能上也有所权衡。

- 对于 `3` 这其实是基于 _在 linuxqq 中的“粘贴”行为读取的是 Wayland 剪贴板而非 X11_ 这个事实，才能起作用的一种方式。其实在我看来这种行为挺奇怪的，尤其是对于一个“同步”脚本来说。

- 但从结果来说，这三种方案中的任意一个都能解决问题，选择哪个其实无所谓。

## clipsync

- 笔者在研究的过程中，正巧 GitHub 上一位用户名为 [SHORiN-KiWATA](https://github.com/SHORiN-KiWATA) 发布了他的[剪贴板同步脚本](https://github.com/SHORiN-KiWATA/clipsync/tree/main)，使用 Shellscript 编写。

- 笔者下载研读了一下，发现他所使用的其实就是上述第一种方法，并且在 md5 哈希计算比对的时候加入了空值检测。

- 观察其 log 可以得到佐证：

    ```plaintext
    [2026-02-14 01:45:38] [INFO] >>> 监测到 X11 剪贴板变化信号
    [2026-02-14 01:45:38] [DEBUG] X11 包含的数据类型: [UTF8_STRING]
    [2026-02-14 01:45:38] [INFO] 匹配类型: 纯文字 (UTF8_STRING)
    [2026-02-14 01:45:38] [DEBUG] 哈希比对: Wayland[8beccf70cb34cb30fcf1b84008cc6d7c] vs X11[221361ceefda8a8097851867a83e2dc1]
    [2026-02-14 01:45:38] [ACTION] 内容不同，正在同步到 Wayland...
    [2026-02-14 01:45:38] [SUCCESS] 纯文字同步成功
    [2026-02-14 01:45:38] [INFO] === 脚本触发: 开始检测剪贴板数据类型 ===
    [2026-02-14 01:45:38] [DEBUG] 检测到的源类型列表: [UTF8_STRING,text/plain,text/plain;charset=utf-8,TEXT,STRING,UTF8_STRING]
    [2026-02-14 01:45:38] [INFO] 匹配类型: 纯文字 (text/plain)
    [2026-02-14 01:45:38] [ACTION] 正在同步纯文字到 X11...
    [2026-02-14 01:45:38] [SUCCESS] 纯文字同步完成
    [2026-02-14 01:45:38] [INFO] --- 流程结束 ---
    [2026-02-14 01:45:38] [INFO] >>> 监测到 X11 剪贴板变化信号
    [2026-02-14 01:45:38] [DEBUG] X11 包含的数据类型: [TARGETS,UTF8_STRING]
    [2026-02-14 01:45:38] [INFO] 匹配类型: 纯文字 (UTF8_STRING)
    [2026-02-14 01:45:38] [DEBUG] 哈希比对: Wayland[221361ceefda8a8097851867a83e2dc1] vs X11[221361ceefda8a8097851867a83e2dc1]
    [2026-02-14 01:45:38] [SKIP] 内容一致，跳过同步
    [2026-02-14 01:45:38] [INFO] --- 本次循环结束，等待下一次 clipnotify 信号 ---
    ```

- 可以看到，一次 X11 剪贴板变化确实触发了 X11 -> Wayland & Wayland -> X11 两次同步，这确实有效的解决了上述问题。

- 同时，使用 systemd 来管理脚本的启停，相较于让脚本自己持有锁来说是一种更好的方式。
- 本着不重复造轮子 ~（懒惰）~ 的原则，笔者决定采用此脚本。

    ```bash
    paru -S clipsync-git
    systemd --user enable clipsync.service --now
    ```

## 另一种方式：XWayland 模式启动

- 既然是由于 linuxqq 对 Wayland 的“表面适配”导致的此等问题，那么直接让 linuxqq 运行在 XWayland 模式下，利用 XWayland 自带的同步机制不就好了？
    - 在 `~/.config` 下创建 `qq-electron-flags.conf`，内容如下：

        ```conf
        --ozone-platform-hint=x11
        ```

- 经测试，XWayland 模式下运行的 linuxqq 没有剪贴板同步问题。

- 或许这才是最简便高效的方式。（摊手）

## 尾声：没有商业价值的用户

- 虽说现如今的 linuxqq 相较于 QQ 的 Electron 时代之前的那个老古董已经完善了不少，但还是在许多细节之处显得非常“敷衍”——不止是剪贴板，语音消息 & 屏幕共享在 Linux 上依旧没有支持。

    > 感觉腾讯全系对桌面 gnu/linux 都是这个态度（
    > 尤其是腾讯会议在 Ubuntu 下二维码都加载不出来，虚拟摄像头也读不到:sweat_smile:

- 该怎么说呢，或许腾讯非常明白他旗下软件的绝大部分用户都不会使用 Linux 作为操作系统，因而压根没投入多少资源。

- 果然 Linux 用户对于国内这些软件厂商来说纯粹是边缘群体，是“没有商业价值的用户”，令人感慨。

## 附录： 关于 linuxqq 的 X11 剪贴板调用

- 使用如下命令启动 linuxqq：

    ```bash
    env -u DISPLAY qq
    ```

- linuxqq 正常启动，但只要在 linuxqq 中右键复制，或者按下 <C-c>，就会发生崩溃报错：

    ```plaintext
    [BuglyService.cpp][handleSignal][382]Stack is succesfully dumped by libUnwind.
    [BuglyService.cpp][handleSignal][384]Native stack:
    $$00    pc 00000000059d41af    /opt/QQ/resources/app/wrapper.node [x86_64::ae125ff74fb6eafb0000000000000000]
    $$01    pc 000000000009698b    /usr/lib/libc.so.6 [x86_64::0f8de86da4eead11213c59d926ca3796]
    $$02    pc 000000000011aa0c    /usr/lib/libc.so.6 [x86_64::0f8de86da4eead11213c59d926ca3796]

    [BuglyService.cpp][handleSignal][386]Record map file of thread: 787
    [BuglyService.cpp][handleSignal][396]Dumping of native stack finished.
    235,0001,Record EupInfo
    235,0000,write key fail
    235,0001,write key fail
    235,0002,write key fail
    235,0003,write key fail
    235,0004,write key fail
    235,0005,write key fail
    235,0006,write key fail
    235,0007,write key fail
    235,0002,Failed to record java thread name.
    235,0008,write key fail
    235,0009,write key fail
    235,0010,write key fail
    235,0003,EupInfo has been recorded.
    235,0004,Record native key-value list.
    235,0005,Native key-value list has been recorded.
    235,0006,Record native log.
    235,0000,Native log has not been initiated.
    235,0007,Native log has been recorded.
    [BuglyService.cpp][clearEupInfo][283]Clear eupInfo object.
    235,0005,Try to unlock file: /home/last/.config/QQ/crash_files//../files/native_record_lock
    235,0006,Successfully unlock file: /home/last/.config/QQ/crash_files//../files/native_record_lock
    [BuglyService.cpp][handleSignal][441]Restored signal handlers.
    ```

## 参考资料

- [X Consortium Standard: Chapter 2 Peer-to-Peer Communication by Means of Selections](https://www.x.org/releases/current/doc/xorg-docs/icccm/icccm.html#Peer_to_Peer_Communication_by_Means_of_Selections)

- [Wikipedia: X Window System selection](https://en.wikipedia.org/wiki/X_Window_System_selection#Clipboard)
