---
title: Linuxqq 粘贴板同步问题
date: 2025-10-15 17:07:43
categories: 工具使用
tags:
    - linuxqq
    - wayland
excerpt: 解决 Linuxqq 中粘贴板无法正常工作的问题
---

## 前言

- 不知从哪个版本开始，Linuxqq 在 Wayland 下的粘贴板出现了一个奇怪的问题：可以将 qq 中的文本复制到系统粘贴板（cliphist），但是无法将系统粘贴板上的内容粘贴进 qq 中。

- 由于问题出现当时我正好将 Linuxqq 换成了 `linuxqq-nt-bwarp`，于是理所当然的认为这是因为 bwarp 沙盒化引起的，乍看之下也挺合理。

- 直到今天在群里看到群主（应要求，不透露 id）发了一份用来同步 `xclip` 和 `cliphist` 粘贴板内容的脚本，这才恍然大悟——沟槽的 tx，虽然 linuxqq 基于的 ELectron 已经默认使用 Wayland 协议，但其复制仍然选择将内容输出到 `xclip`。

## 脚本

- 脚本代码如下：

    {% folding blue:: clipboard_sync.sh %}

    ```bash
    #!/bin/bash

    # Lock file to prevent multiple instances
    LOCK_FILE="/tmp/clipboard_sync.lock"

    exec 200>"$LOCK_FILE"
    flock -n 200 || {
        echo "Error: Another instance of the script is already running." >&2
        exit 1
    }

    # The interval in seconds for checking the clipboard
    INTERVAL=1

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
                if [[ $(xclip -selection clipboard -t "$x11_img_type" -o 2>/dev/null | wc -c) -gt 0 ]]; then
                    x11_img_hash=$(xclip -selection clipboard -t "$x11_img_type" -o 2>/dev/null | sha256sum | awk '{print $1}')

                    if [[ -n "$x11_img_hash" && "$x11_img_hash" != "$last_x11_img_hash" ]]; then
                        # Pipe the image data directly to the Wayland clipboard
                        xclip -selection clipboard -t "$x11_img_type" -o 2>/dev/null | wl-copy -t "$x11_img_type"
                        last_x11_img_hash="$x11_img_hash"
                        last_wl_img_hash="$x11_img_hash"
                        img_synced=true
                        log_sync "x11->wl" "Image" " ($x11_img_type)"
                        # Clear the X11 clipboard to make Wayland the single source of truth and prevent echo-syncing
                        xclip -selection clipboard -i /dev/null
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
                    wl_img_hash=$(wl-paste -t "$wl_img_type" 2>/dev/null | tee >(sha256sum | awk '{print $1}' > /tmp/wl_hash_$$) | cat > /dev/null; cat /tmp/wl_hash_$$ 2>/dev/null; rm -f /tmp/wl_hash_$$ 2>/dev/null)

                    # If the Wayland image is different from the last synced one, sync it to X11
                    if [[ -n "$wl_img_hash" && "$wl_img_hash" != "$last_wl_img_hash" ]]; then
                        wl-paste -t "$wl_img_type" 2>/dev/null | xclip -selection clipboard -t "$wl_img_type" -i
                        last_wl_img_hash="$wl_img_hash"
                        last_x11_img_hash="$wl_img_hash"
                        img_synced=true  # Mark image as synced to skip text sync in this iteration
                        log_sync "wl->x11" "Image" " ($wl_img_type)"
                    fi
                fi
            fi

            # -------- Text Sync --------
            # Only perform text sync if no image was synced in this iteration
            if [[ "$img_synced" == false ]]; then
                text_synced=false  # Flag to track if text was synced in this iteration
                current_text=$(wl-paste --type text/plain 2>/dev/null || true)

                # Only read from X11 clipboard if it likely contains text to avoid warnings from binary data
                x11_text=""
                if [[ -z "$x11_img_type" ]]; then
                    x11_text=$(xclip -selection clipboard -o 2>/dev/null || true)
                fi

                if [[ -n "$x11_text" && "$x11_text" != "$last_text" && "$x11_text" != "$current_text" ]]; then
                    echo -n "$x11_text" | wl-copy --type text/plain
                    last_text="$x11_text"
                    text_synced=true
                    # Show a text preview (max 50 chars), replacing special characters
                    local preview=$(echo -n "$x11_text" | head -c 50 | iconv -f utf-8 -t utf-8 -c)
                    preview="${preview//$'\n'/↵}"
                    preview="${preview//$'\r'/}"
                    preview="${preview//$'\t'/⇥}"
                    [[ ${#x11_text} -gt 50 ]] && preview="${preview}..."
                    log_sync "x11->wl" "Text" " \"$preview\""
                fi

                # Only check Wayland → X11 if no text sync has occurred from the other direction
                if [[ "$text_synced" == false && -n "$current_text" && "$current_text" != "$last_text" && "$x11_text" != "$current_text" ]]; then
                    echo "$current_text" | xclip -selection clipboard -t UTF8_STRING -i
                    last_text="$current_text"
                    # Show a text preview (max 50 chars), replacing special characters
                    local preview=$(echo -n "$current_text" | head -c 50 | iconv -f utf-8 -t utf-8 -c)
                    preview="${preview//$'\n'/↵}"
                    preview="${preview//$'\r'/}"
                    preview="${preview//$'\t'/⇥}"
                    [[ ${#current_text} -gt 50 ]] && preview="${preview}..."
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

    # Start the sync service
    clipboard_sync
    ```

    {% endfolding %}

---

- 这个脚本的思路很简单：每间隔 `$INTERVAL` 的时间，就同步一次两个剪贴板中的内容。同时，利用哈希算法来确保图片的同步过程中不会出现无限循环。

- 同时，使用 `flock` 来确保不会出现该脚本在同一时间被运行多次。

    ```bash
    LOCK_FILE="/tmp/clipboard_sync.lock"

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

