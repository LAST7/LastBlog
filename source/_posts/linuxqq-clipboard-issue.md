---
title: Linuxqq 粘贴板同步问题
date: 2025-10-15 17:07:43
categories: 工具使用
tags:
    - linuxqq
    - wayland
    - shell
excerpt: 解决 Linuxqq 在 Wayland 协议下粘贴板无法正常工作的问题
---

{% notel red fa-clock **更新** %}

- 该 Shell 脚本因改来改去 Bug 频出，因此近日笔者细细研究了一番这其中的古怪机制，详情见 [这篇博文](https://blog.imlast.top/2026/02/13/clipboard-sync/)

{% endnotel %}

## 前言

- 不知从哪个版本开始，Linuxqq 在 Wayland 下的粘贴板出现了一个奇怪的问题：可以将 qq 中的文本复制到系统粘贴板（cliphist），但是无法将系统粘贴板上的内容粘贴进 qq 中。

- 由于问题出现当时我正好将 Linuxqq 换成了 `linuxqq-nt-bwarp`，于是理所当然的认为这是因为 bwarp 沙盒化引起的，乍看之下也挺合理。

- 直到今天在群里看到群主（应要求，不透露 id）发了一份用来同步 `xclip` 和 `cliphist` 粘贴板内容的脚本，这才恍然大悟——沟槽的 tx，虽然 linuxqq 基于的 Electron 已经默认使用 Wayland 协议，但其复制仍然选择将内容输出到 `xclip`。

---

{% notel blue fa-circle-info 查看 xclip %}
可以使用 `xclip -sel clip -o` 来查看 xlip 粘贴板中的内容。
{% endnotel %}

## 脚本

- 脚本代码如下：

    {% folding blue:: clipboard_sync.sh %}

    ```bash
    #!/bin/bash

    # Lock file to prevent multiple instances
    LOCK_DIR="${XDG_RUNTIME_DIR:-/tmp}"
    LOCK_FILE="$LOCK_DIR/clipboard_sync.lock"

    exec 200>"$LOCK_FILE"
    flock -n 200 || {
        echo "Error: Another instance of the script is already running." >&2
        exit 1
    }

    # Temp file path
    TEMP_IMG="$LOCK_DIR/clip_sync_buffer"

    # The interval in seconds for checking the clipboard
    INTERVAL=0.5

    # Color definitions for logging
    COLOR_RESET="\033[0m"
    COLOR_GREEN="\033[32m"
    COLOR_BLUE="\033[34m"
    COLOR_YELLOW="\033[33m"
    COLOR_CYAN="\033[36m"

    # General log function
    log() {
        local timestamp=$(date '+%H:%M:%S')
        echo -e "${COLOR_CYAN}[$timestamp]${COLOR_RESET} $1"
    }

    # Log function specific to sync events
    log_sync() {
        local direction=$1
        local type=$2
        local format=$3

        case "$direction" in
            "x11->wl")
                echo -e "${COLOR_CYAN}[$(date '+%H:%M:%S')]${COLOR_RESET} ${COLOR_GREEN}✓${COLOR_RESET} X11 → Wayland | ${COLOR_YELLOW}${type}${COLOR_RESET}${format}"
                ;;
            "wl->x11")
                echo -e "${COLOR_CYAN}[$(date '+%H:%M:%S')]${COLOR_RESET} ${COLOR_BLUE}✓${COLOR_RESET} Wayland → X11 | ${COLOR_YELLOW}${type}${COLOR_RESET}${format}"
                ;;
        esac
    }

    # Function to check for required command dependencies
    check_dependencies() {
        local missing_deps=()
        for cmd in xclip wl-paste wl-copy sha256sum; do
            if ! command -v "$cmd" &>/dev/null; then
                missing_deps+=("$cmd")
            fi
        done

        if [[ ${#missing_deps[@]} -gt 0 ]]; then
            echo "Error: Missing required dependencies: ${missing_deps[*]}" >&2
            exit 1
        fi
    }

    # Debounce variables
    last_text=""
    last_x11_img_hash=""  # Hash of the last image synced from X11 to Wayland
    last_wl_img_hash=""   # Hash of the last image synced from Wayland to X11

    # Main sync loop
    clipboard_sync() {
        while true; do
            img_synced=false  # Flag to track if an image was synced in this iteration

            # -------- Image Sync: X11 → Wayland --------
            x11_targets=$(xclip -selection clipboard -t TARGETS -o 2>/dev/null || true)
            x11_img_type=""
            if echo "$x11_targets" | grep -q "image/png"; then
                x11_img_type="image/png"
            elif echo "$x11_targets" | grep -q "image/jpeg"; then
                x11_img_type="image/jpeg"
            elif echo "$x11_targets" | grep -q "image/gif"; then
                x11_img_type="image/gif"
            fi

            if [[ -n "$x11_img_type" ]]; then
                if xclip -selection clipboard -t "$x11_img_type" -o > "TEMP_IMG" 2>/dev/null; then

                    # image output file exists
                    if [[ -s "$TEMP_IMG" ]]; then
                        x11_img_hash=$(xclip -selection clipboard -t "$x11_img_type" -o 2>/dev/null | sha256sum | awk '{print $1}')

                        if [[ -n "$x11_img_hash" && "$x11_img_hash" != "$last_x11_img_hash" ]]; then

                            # use temp file as buffer instead of using pipeline directly to avoid image cut off
                            wl-copy -t "$x11_img_type" < "$TEMP_IMG"

                            last_x11_img_hash="$x11_img_hash"
                            last_wl_img_hash="$x11_img_hash"
                            img_synced=true
                            log_sync "x11->wl" "Image" " ($x11_img_type)"
                        fi
                    fi

                fi
            fi

            # -------- Image Sync: Wayland → X11 --------
            # Only check Wayland → X11 if no image sync has occurred in this iteration
            if [[ "$img_synced" == false ]]; then
                wl_types=$(wl-paste --list-types 2>/dev/null || true)
                wl_img_type=""
                if echo "$wl_types" | grep -q "image/png"; then
                    wl_img_type="image/png"
                elif echo "$wl_types" | grep -q "image/jpeg"; then
                    wl_img_type="image/jpeg"
                elif echo "$wl_types" | grep -q "image/gif"; then
                    wl_img_type="image/gif"
                fi

                if [[ -n "$wl_img_type" ]]; then
                    # Pipe binary data to calculate hash for debouncing
                    # wl_img_hash=$(wl-paste -t "$wl_img_type" 2>/dev/null | tee >(sha256sum | awk '{print $1}' > /tmp/wl_hash_$$) | cat > /dev/null; cat /tmp/wl_hash_$$ 2>/dev/null; rm -f /tmp/wl_hash_$$ 2>/dev/null)

                    if wl-paste -t "$wl_img_type" > "$TEMP_IMG" 2>/dev/null; then
                        if [[ -s "$TEMP_IMG" ]]; then
                            wl_img_hash=$(sha256sum "$TEMP_IMG" | awk '{print $1}')

                            # If the Wayland image is different from the last synced one, sync it to X11
                            if [[ -n "$wl_img_hash" && "$wl_img_hash" != "$last_wl_img_hash" ]]; then
                                xclip -selection clipboard -t "wl_img_type" -i < "$TEMP_IMG"

                                last_wl_img_hash="$wl_img_hash"
                                last_x11_img_hash="$wl_img_hash"
                                img_synced=true  # Mark image as synced to skip text sync in this iteration
                                log_sync "wl->x11" "Image" " ($wl_img_type)"
                            fi
                        fi
                    fi

                fi
            fi

            # -------- Text Sync --------
            # Only perform text sync if no image was synced in this iteration
            if [[ "$img_synced" == false ]]; then
                text_synced=false  # Flag to track if text was synced in this iteration

                x11_targets=$(xclip -selection clipboard -t TARGETS -o 2>/dev/null || true)

                # Only read from X11 clipboard if it likely contains text to avoid warnings from binary data
                if echo "x11_targets" | grep -qE "image/png|image/jpeg|image/bmp|image/tiff"; then
                    x11_text=""
                else
                    x11_text=$(xclip -selection clipboard -t UTF8_STRING -o 2>/dev/null || true)
                fi

                current_text=$(wl-paste --type text/plain 2>/dev/null || true)

                if [[ -n "$x11_text" && "$x11_text" != "$last_text" && "$x11_text" != "$current_text" ]]; then
                    if [[ ! "$x11_text" =~ \x00 ]]; then
                        # use `printf` for special characters
                        printf "%s" "$x11_text" | wl-copy --type text/plain
                        last_text="$x11_text"
                        text_synced=true
                        log_sync "x11->wl" "Text" " \"$preview\""
                    fi
                fi

                # Only check Wayland → X11 if no text sync has occurred from the other direction
                if [[ "$text_synced" == false && -n "$current_text" && "$current_text" != "$last_text" && "$x11_text" != "$current_text" ]]; then
                    # use `printf` for special characters
                    printf "%s" "$current_text" | xclip -selection clipboard -t UTF8_STRING -i
                    last_text="$current_text"
                    log_sync "wl->x11" "Text" " \"$preview\""
                fi
            fi

            sleep $INTERVAL
        done
    }

    # Set up cleanup on exit (releases the lock file)
    trap 'exit' INT TERM EXIT

    # Check for dependencies
    check_dependencies

    log "Clipboard sync service started"
    log "Monitoring clipboard sync between Wayland ↔ X11..."

    log "Service started on lock: $LOCK_FILE"

    # Start the sync service
    clipboard_sync
    ```

{% endfolding %}

- 此前的脚本使用管道来传输图片文件，这个操作实际上是异步的。执行管道传输命令之后直接清空 xclip 中的内容，就可能导致同步的图片出现被“截断”的现象。

- 操作稍有复杂的脚本还是应该高级语言写……

{% endnotel %}

## 前言

- 不知从哪个版本开始，Linuxqq 在 Wayland 下的粘贴板出现了一个奇怪的问题：可以将 qq 中的文本复制到系统粘贴板（cliphist），但是无法将系统粘贴板上的内容粘贴进 qq 中。

- 由于问题出现当时我正好将 Linuxqq 换成了 `linuxqq-nt-bwarp`，于是理所当然的认为这是因为 bwarp 沙盒化引起的，乍看之下也挺合理。

- 直到今天在群里看到群主（应要求，不透露 id）发了一份用来同步 `xclip` 和 `cliphist` 粘贴板内容的脚本，这才恍然大悟——沟槽的 tx，虽然 linuxqq 基于的 Electron 已经默认使用 Wayland 协议，但其复制仍然选择将内容输出到 `xclip`。

---

{% notel blue fa-circle-info 查看 xclip %}
可以使用 `xclip -sel clip -o` 来查看 xlip 粘贴板中的内容。
{% endnotel %}

## 脚本

- 脚本代码如下：

    {% folding blue:: clipboard_sync.sh %}

    ```bash
    #!/bin/bash

    # Lock file to prevent multiple instances
    LOCK_DIR="${XDG_RUNTIME_DIR:-/tmp}"
    LOCK_FILE="$LOCK_DIR/clipboard_sync.lock"

    exec 200>"$LOCK_FILE"
    flock -n 200 || {
        echo "Error: Another instance of the script is already running." >&2
        exit 1
    }

    # Temp file path
    TEMP_IMG="$LOCK_DIR/clip_sync_buffer"

    # The interval in seconds for checking the clipboard
    INTERVAL=0.5

    # Color definitions for logging
    COLOR_RESET="\033[0m"
    COLOR_GREEN="\033[32m"
    COLOR_BLUE="\033[34m"
    COLOR_YELLOW="\033[33m"
    COLOR_CYAN="\033[36m"

    # General log function
    log() {
        local timestamp=$(date '+%H:%M:%S')
        echo -e "${COLOR_CYAN}[$timestamp]${COLOR_RESET} $1"
    }

    # Log function specific to sync events
    log_sync() {
        local direction=$1
        local type=$2
        local format=$3

        case "$direction" in
            "x11->wl")
                echo -e "${COLOR_CYAN}[$(date '+%H:%M:%S')]${COLOR_RESET} ${COLOR_GREEN}✓${COLOR_RESET} X11 → Wayland | ${COLOR_YELLOW}${type}${COLOR_RESET}${format}"
                ;;
            "wl->x11")
                echo -e "${COLOR_CYAN}[$(date '+%H:%M:%S')]${COLOR_RESET} ${COLOR_BLUE}✓${COLOR_RESET} Wayland → X11 | ${COLOR_YELLOW}${type}${COLOR_RESET}${format}"
                ;;
        esac
    }

    # Function to check for required command dependencies
    check_dependencies() {
        local missing_deps=()
        for cmd in xclip wl-paste wl-copy sha256sum; do
            if ! command -v "$cmd" &>/dev/null; then
                missing_deps+=("$cmd")
            fi
        done

        if [[ ${#missing_deps[@]} -gt 0 ]]; then
            echo "Error: Missing required dependencies: ${missing_deps[*]}" >&2
            exit 1
        fi
    }

    # Debounce variables
    last_text=""
    last_x11_img_hash=""  # Hash of the last image synced from X11 to Wayland
    last_wl_img_hash=""   # Hash of the last image synced from Wayland to X11

    # Main sync loop
    clipboard_sync() {
        while true; do
            img_synced=false  # Flag to track if an image was synced in this iteration

            # -------- Image Sync: X11 → Wayland --------
            x11_targets=$(xclip -selection clipboard -t TARGETS -o 2>/dev/null || true)
            x11_img_type=""
            if echo "$x11_targets" | grep -q "image/png"; then
                x11_img_type="image/png"
            elif echo "$x11_targets" | grep -q "image/jpeg"; then
                x11_img_type="image/jpeg"
            elif echo "$x11_targets" | grep -q "image/gif"; then
                x11_img_type="image/gif"
            fi

            if [[ -n "$x11_img_type" ]]; then
                if xclip -selection clipboard -t "$x11_img_type" -o > "TEMP_IMG" 2>/dev/null; then

                    # image output file exists
                    if [[ -s "$TEMP_IMG" ]]; then
                        x11_img_hash=$(xclip -selection clipboard -t "$x11_img_type" -o 2>/dev/null | sha256sum | awk '{print $1}')

                        if [[ -n "$x11_img_hash" && "$x11_img_hash" != "$last_x11_img_hash" ]]; then

                            # use temp file as buffer instead of using pipeline directly to avoid image cut off
                            wl-copy -t "$x11_img_type" < "$TEMP_IMG"

                            last_x11_img_hash="$x11_img_hash"
                            last_wl_img_hash="$x11_img_hash"
                            img_synced=true
                            log_sync "x11->wl" "Image" " ($x11_img_type)"
                        fi
                    fi

                fi
            fi

            # -------- Image Sync: Wayland → X11 --------
            # Only check Wayland → X11 if no image sync has occurred in this iteration
            if [[ "$img_synced" == false ]]; then
                wl_types=$(wl-paste --list-types 2>/dev/null || true)
                wl_img_type=""
                if echo "$wl_types" | grep -q "image/png"; then
                    wl_img_type="image/png"
                elif echo "$wl_types" | grep -q "image/jpeg"; then
                    wl_img_type="image/jpeg"
                elif echo "$wl_types" | grep -q "image/gif"; then
                    wl_img_type="image/gif"
                fi

                if [[ -n "$wl_img_type" ]]; then
                    # Pipe binary data to calculate hash for debouncing
                    # wl_img_hash=$(wl-paste -t "$wl_img_type" 2>/dev/null | tee >(sha256sum | awk '{print $1}' > /tmp/wl_hash_$$) | cat > /dev/null; cat /tmp/wl_hash_$$ 2>/dev/null; rm -f /tmp/wl_hash_$$ 2>/dev/null)

                    if wl-paste -t "$wl_img_type" > "$TEMP_IMG" 2>/dev/null; then
                        if [[ -s "$TEMP_IMG" ]]; then
                            wl_img_hash=$(sha256sum "$TEMP_IMG" | awk '{print $1}')

                            # If the Wayland image is different from the last synced one, sync it to X11
                            if [[ -n "$wl_img_hash" && "$wl_img_hash" != "$last_wl_img_hash" ]]; then
                                xclip -selection clipboard -t "wl_img_type" -i < "$TEMP_IMG"

                                last_wl_img_hash="$wl_img_hash"
                                last_x11_img_hash="$wl_img_hash"
                                img_synced=true  # Mark image as synced to skip text sync in this iteration
                                log_sync "wl->x11" "Image" " ($wl_img_type)"
                            fi
                        fi
                    fi

                fi
            fi

            # -------- Text Sync --------
            # Only perform text sync if no image was synced in this iteration
            if [[ "$img_synced" == false ]]; then
                text_synced=false  # Flag to track if text was synced in this iteration

                x11_targets=$(xclip -selection clipboard -t TARGETS -o 2>/dev/null || true)

                # Only read from X11 clipboard if it likely contains text to avoid warnings from binary data
                if echo "x11_targets" | grep -qE "image/png|image/jpeg|image/bmp|image/tiff"; then
                    x11_text=""
                else
                    x11_text=$(xclip -selection clipboard -t UTF8_STRING -o 2>/dev/null || true)
                fi

                current_text=$(wl-paste --type text/plain 2>/dev/null || true)

                if [[ -n "$x11_text" && "$x11_text" != "$last_text" && "$x11_text" != "$current_text" ]]; then
                    if [[ ! "$x11_text" =~ \x00 ]]; then
                        # use `printf` for special characters
                        printf "%s" "$x11_text" | wl-copy --type text/plain
                        last_text="$x11_text"
                        text_synced=true
                        log_sync "x11->wl" "Text" " \"$preview\""
                    fi
                fi

                # Only check Wayland → X11 if no text sync has occurred from the other direction
                if [[ "$text_synced" == false && -n "$current_text" && "$current_text" != "$last_text" && "$x11_text" != "$current_text" ]]; then
                    # use `printf` for special characters
                    printf "%s" "$current_text" | xclip -selection clipboard -t UTF8_STRING -i
                    last_text="$current_text"
                    log_sync "wl->x11" "Text" " \"$preview\""
                fi
            fi

            sleep $INTERVAL
        done
    }

    # Set up cleanup on exit (releases the lock file)
    trap 'exit' INT TERM EXIT

    # Check for dependencies
    check_dependencies

    log "Clipboard sync service started"
    log "Monitoring clipboard sync between Wayland ↔ X11..."

    log "Service started on lock: $LOCK_FILE"

    # Start the sync service
    clipboard_sync
    ```

    {% endfolding %}

---

- 这个脚本的思路很简单：每间隔 `$INTERVAL` 的时间，就同步一次两个剪贴板中的内容。同时，利用哈希算法来确保图片的同步过程中不会出现无限循环。

- 同时，笔者加入了使用 `flock` 的片段来确保不会出现该脚本在同一时间被运行多次的情况。

    ```bash
    # Lock file to prevent multiple instances
    LOCK_DIR="${XDG_RUNTIME_DIR:-/tmp}"
    LOCK_FILE="$LOCK_DIR/clipboard_sync.lock"

    exec 200>"$LOCK_FILE"
    flock -n 200 || {
        echo "Error: Another instance of the script is already running." >&2
        exit 1
    }

    # ...


    trap 'exit' INT TERM EXIT
    ```

## 后台运行

- 笔者使用的是 Hyprland，因此只需在配置文件中加入：

    ```conf
    # Start clipboard sync service (for linuxqq)
    exec = ~/path/to/clipboard_sync.sh
    ```

- 即可。

```

```
