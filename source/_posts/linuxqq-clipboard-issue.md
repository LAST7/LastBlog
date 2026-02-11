---
title: Linuxqq 粘贴板同步问题
date: 2025-10-15 17:07:43
categories: 工具使用
tags:
    - linuxqq
    - wayland
excerpt: 解决 Linuxqq 在 Wayland 协议下粘贴板无法正常工作的问题
---

{% notel red fa-clock **更新** %}

- 该 Shell 脚本因改来改去 Bug 频出，因此近日笔者又让 LLM 照着相同的逻辑搓了一个 Python 版本的脚本，如有问题欢迎留言。

{% folding blue:: clipboard_sync.sh %}

```python
#!/usr/bin/env python3
"""
Clipboard sync between Wayland and X11
"""

import subprocess
import hashlib
import time
import sys
import signal
import os
from pathlib import Path
from typing import Optional, Tuple
import logging

# ================= Configuration =================

SYNC_INTERVAL = 0.3
LOCK_FILE = Path(os.environ.get("XDG_RUNTIME_DIR", "/tmp")) / "clipboard_sync.lock"
LOG_FORMAT = "%(asctime)s [%(levelname)s] %(message)s"
LOG_TIME_FORMAT = "%H:%M:%S"

# ================= Logging =================


# Custom colored formatter for better readability
class ColoredFormatter(logging.Formatter):
    COLORS = {
        "INFO": "\033[36m",  # Cyan
        "SUCCESS_X11": "\033[32m",  # Green
        "SUCCESS_WL": "\033[34m",  # Blue
        "RESET": "\033[0m",
    }

    def format(self, record):
        if record.levelname == "INFO":
            # Add colored checkmarks for sync events
            if "X11 → Wayland" in record.msg:
                record.msg = (
                    f"{self.COLORS['SUCCESS_X11']}✓{self.COLORS['RESET']} {record.msg}"
                )
            elif "Wayland → X11" in record.msg:
                record.msg = (
                    f"{self.COLORS['SUCCESS_WL']}✓{self.COLORS['RESET']} {record.msg}"
                )
        return super().format(record)


# Setup logger with colored output
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)
handler = logging.StreamHandler()
handler.setFormatter(ColoredFormatter(LOG_FORMAT, datefmt=LOG_TIME_FORMAT))
logger.addHandler(handler)

# ================= Lock Mechanism =================


def acquire_lock():
    """Acquire singleton lock to prevent multiple instances"""
    try:
        # Open file in exclusive mode
        fd = os.open(str(LOCK_FILE), os.O_CREAT | os.O_EXCL | os.O_WRONLY, 0o644)
        os.close(fd)
        return True
    except FileExistsError:
        logger.error("Another instance is already running.")
        return False


def release_lock():
    """Release the lock file"""
    try:
        LOCK_FILE.unlink()
    except FileNotFoundError:
        pass


# ================= Clipboard Operations =================


class ClipboardManager:
    """Manages X11 and Wayland clipboard operations"""

    # Supported image MIME types
    IMAGE_TYPES = ["image/png", "image/jpeg", "image/bmp", "image/tiff"]

    # Known types that we handle or can safely ignore
    KNOWN_TYPES = {
        "image/png",
        "image/jpeg",
        "image/bmp",
        "image/tiff",
        "text/plain",
        "UTF8_STRING",
        "STRING",
        "TEXT",
        "TARGETS",
        "TIMESTAMP",
        "SAVE_TARGETS",
        "MULTIPLE",
        "text/plain;charset=utf-8",
    }

    @staticmethod
    def run_cmd(cmd: list, input_data: Optional[bytes] = None) -> Tuple[bool, bytes]:
        """
        Run a shell command and return (success, output)

        Args:
            cmd: Command and arguments as a list
            input_data: Optional bytes to pipe to stdin

        Returns:
            Tuple of (success: bool, output: bytes)
        """
        try:
            result = subprocess.run(
                cmd,
                input=input_data,
                capture_output=True,
                timeout=2,
            )
            return result.returncode == 0, result.stdout
        except (subprocess.TimeoutExpired, FileNotFoundError):
            return False, b""

    @staticmethod
    def is_valid_text(data: bytes) -> bool:
        """
        Check if data is valid text (not binary garbage)

        Validates by:
        1. Checking for null bytes (common in binary data)
        2. Ensuring at least 80% of characters are printable

        Args:
            data: Raw bytes to validate

        Returns:
            True if data appears to be valid text
        """
        if b"\x00" in data:
            return False

        # Check ratio of printable characters
        try:
            text = data.decode("utf-8", errors="ignore")
            printable = sum(1 for c in text if c.isprintable() or c.isspace())
            return printable >= len(text) * 0.8  # At least 80% printable
        except:
            return False

    # -------- X11 Operations --------

    @staticmethod
    def x11_get_targets() -> list:
        """Get list of MIME types available in X11 clipboard"""
        success, output = ClipboardManager.run_cmd(
            ["xclip", "-selection", "clipboard", "-t", "TARGETS", "-o"]
        )
        if not success:
            return []
        return output.decode("utf-8", errors="ignore").strip().split("\n")

    @staticmethod
    def x11_get_image(mime_type: str) -> Optional[bytes]:
        """Read image data from X11 clipboard with specified MIME type"""
        success, data = ClipboardManager.run_cmd(
            ["xclip", "-selection", "clipboard", "-t", mime_type, "-o"]
        )
        return data if success and len(data) > 0 else None

    @staticmethod
    def x11_get_text() -> Optional[str]:
        """Read text from X11 clipboard"""
        success, data = ClipboardManager.run_cmd(
            ["xclip", "-selection", "clipboard", "-t", "UTF8_STRING", "-o"]
        )
        if success and ClipboardManager.is_valid_text(data):
            return data.decode("utf-8", errors="ignore")
        return None

    @staticmethod
    def x11_set_image(mime_type: str, data: bytes):
        """Write image data to X11 clipboard"""
        ClipboardManager.run_cmd(
            ["xclip", "-selection", "clipboard", "-t", mime_type, "-i"],
            input_data=data,
        )

    @staticmethod
    def x11_set_text(text: str):
        """Write text to X11 clipboard"""
        ClipboardManager.run_cmd(
            ["xclip", "-selection", "clipboard", "-t", "UTF8_STRING", "-i"],
            input_data=text.encode("utf-8"),
        )

    # -------- Wayland Operations --------

    @staticmethod
    def wl_get_types() -> list:
        """Get list of MIME types available in Wayland clipboard"""
        success, output = ClipboardManager.run_cmd(["wl-paste", "--list-types"])
        if not success:
            return []
        return output.decode("utf-8", errors="ignore").strip().split("\n")

    @staticmethod
    def wl_get_image(mime_type: str) -> Optional[bytes]:
        """Read image data from Wayland clipboard with specified MIME type"""
        success, data = ClipboardManager.run_cmd(["wl-paste", "-t", mime_type])
        return data if success and len(data) > 0 else None

    @staticmethod
    def wl_get_text() -> Optional[str]:
        """Read text from Wayland clipboard"""
        success, data = ClipboardManager.run_cmd(["wl-paste", "--type", "text/plain"])
        if success and ClipboardManager.is_valid_text(data):
            return data.decode("utf-8", errors="ignore")
        return None

    @staticmethod
    def wl_set_image(mime_type: str, data: bytes):
        """Write image data to Wayland clipboard"""
        ClipboardManager.run_cmd(
            ["wl-copy", "-t", mime_type],
            input_data=data,
        )

    @staticmethod
    def wl_set_text(text: str):
        """Write text to Wayland clipboard"""
        ClipboardManager.run_cmd(
            ["wl-copy", "--type", "text/plain"],
            input_data=text.encode("utf-8"),
        )


# ================= Sync Logic =================


class ClipboardSync:
    """Handles bidirectional clipboard synchronization between X11 and Wayland"""

    def __init__(self):
        # State tracking for deduplication
        self.last_text = ""
        self.last_x11_img_hash = ""
        self.last_wl_img_hash = ""
        self.last_img_sync_time = 0

        # Track unhandled types for debugging
        self.logged_unhandled_types = set()

    @staticmethod
    def hash_data(data: bytes) -> str:
        """Calculate SHA256 hash of data for deduplication"""
        return hashlib.sha256(data).hexdigest()

    def log_sync(self, direction: str, content_type: str, detail: str = ""):
        """Log a sync event with direction and content type"""
        if direction == "x11->wl":
            logger.info(f"X11 → Wayland | {content_type}{detail}")
        elif direction == "wl->x11":
            logger.info(f"Wayland → X11 | {content_type}{detail}")

    def log_unhandled_types(self, x11_types: list, wl_types: list):
        """
        Log any clipboard MIME types that we don't currently handle.
        This helps identify if users need support for additional formats.
        Only logs each unique type once to avoid spam.
        """
        all_types = set(x11_types + wl_types)
        # Filter out empty strings and known types
        unhandled = {
            t for t in all_types if t and t not in ClipboardManager.KNOWN_TYPES
        }

        # Only log new unhandled types
        new_unhandled = unhandled - self.logged_unhandled_types
        if new_unhandled:
            logger.debug(
                f"Unhandled clipboard types detected: {', '.join(sorted(new_unhandled))}"
            )
            self.logged_unhandled_types.update(new_unhandled)

    def sync_image_x11_to_wl(self) -> bool:
        """
        Sync image from X11 to Wayland clipboard

        Returns:
            True if an image was synced, False otherwise
        """
        targets = ClipboardManager.x11_get_targets()

        # Find first supported image format
        img_type = None
        for mime in ClipboardManager.IMAGE_TYPES:
            if mime in targets:
                img_type = mime
                break

        if not img_type:
            return False

        img_data = ClipboardManager.x11_get_image(img_type)
        if not img_data:
            return False

        img_hash = self.hash_data(img_data)

        # Deduplicate: skip if already synced
        if img_hash == self.last_x11_img_hash:
            return False

        # Sync to Wayland
        ClipboardManager.wl_set_image(img_type, img_data)
        self.last_x11_img_hash = img_hash
        self.last_wl_img_hash = img_hash
        self.last_img_sync_time = time.time()
        self.log_sync("x11->wl", "Image", f" ({img_type})")
        return True

    def sync_image_wl_to_x11(self) -> bool:
        """
        Sync image from Wayland to X11 clipboard

        Returns:
            True if an image was synced, False otherwise
        """
        types = ClipboardManager.wl_get_types()

        # Find first supported image format
        img_type = None
        for mime in ClipboardManager.IMAGE_TYPES:
            if mime in types:
                img_type = mime
                break

        if not img_type:
            return False

        img_data = ClipboardManager.wl_get_image(img_type)
        if not img_data:
            return False

        img_hash = self.hash_data(img_data)

        # Deduplicate: skip if already synced
        if img_hash == self.last_wl_img_hash:
            return False

        # Sync to X11
        ClipboardManager.x11_set_image(img_type, img_data)
        self.last_wl_img_hash = img_hash
        self.last_x11_img_hash = img_hash
        self.last_img_sync_time = time.time()
        self.log_sync("wl->x11", "Image", f" ({img_type})")
        return True

    def sync_text_x11_to_wl(self) -> bool:
        """
        Sync text from X11 to Wayland clipboard

        Returns:
            True if text was synced, False otherwise
        """
        # Skip if X11 clipboard contains images
        targets = ClipboardManager.x11_get_targets()
        if any(img in targets for img in ClipboardManager.IMAGE_TYPES):
            return False

        text = ClipboardManager.x11_get_text()
        if not text or text == self.last_text:
            return False

        # Check if Wayland already has the same content
        wl_text = ClipboardManager.wl_get_text()
        if text == wl_text:
            return False

        ClipboardManager.wl_set_text(text)
        self.last_text = text
        preview = text[:50] + ("..." if len(text) > 50 else "")
        self.log_sync("x11->wl", "Text", f' "{preview}"')
        return True

    def sync_text_wl_to_x11(self) -> bool:
        """
        Sync text from Wayland to X11 clipboard

        Returns:
            True if text was synced, False otherwise
        """
        # Skip if Wayland clipboard contains images
        types = ClipboardManager.wl_get_types()
        if any(img in types for img in ClipboardManager.IMAGE_TYPES):
            return False

        text = ClipboardManager.wl_get_text()
        if not text or text == self.last_text:
            return False

        # Check if X11 already has the same content
        x11_text = ClipboardManager.x11_get_text()
        if text == x11_text:
            return False

        ClipboardManager.x11_set_text(text)
        self.last_text = text
        preview = text[:50] + ("..." if len(text) > 50 else "")
        self.log_sync("wl->x11", "Text", f' "{preview}"')
        return True

    def run(self):
        """Main sync loop - continuously monitors and syncs clipboard content"""
        logger.info("Clipboard sync service started")
        logger.info("Monitoring clipboard sync between Wayland ↔ X11...")

        try:
            while True:
                # Get current clipboard types for unhandled type detection
                x11_types = ClipboardManager.x11_get_targets()
                wl_types = ClipboardManager.wl_get_types()

                # Log any unhandled types (for debugging/future feature requests)
                self.log_unhandled_types(x11_types, wl_types)

                # Priority 1: Sync images (bidirectional)
                img_synced = self.sync_image_x11_to_wl() or self.sync_image_wl_to_x11()

                # Priority 2: Sync text (only if no image was synced)
                if not img_synced:
                    # Cooldown period after image sync to prevent race conditions
                    if time.time() - self.last_img_sync_time < 0.4:
                        time.sleep(0.1)
                    else:
                        self.sync_text_x11_to_wl() or self.sync_text_wl_to_x11()

                time.sleep(SYNC_INTERVAL)
        except KeyboardInterrupt:
            logger.info("Shutting down...")


# ================= Main Entry Point =================


def main():
    """Main entry point - checks dependencies, acquires lock, and starts sync"""
    # Check for required dependencies
    for cmd in ["xclip", "wl-paste", "wl-copy"]:
        if subprocess.run(["which", cmd], capture_output=True).returncode != 0:
            logger.error(f"Missing dependency: {cmd}")
            sys.exit(1)

    # Acquire singleton lock
    if not acquire_lock():
        sys.exit(1)

    # Setup signal handlers for clean shutdown
    def signal_handler(sig, frame):
        release_lock()
        sys.exit(0)

    signal.signal(signal.SIGINT, signal_handler)
    signal.signal(signal.SIGTERM, signal_handler)

    # Start clipboard sync
    sync = ClipboardSync()
    sync.run()


if __name__ == "__main__":
    main()
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
