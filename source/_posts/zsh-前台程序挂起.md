---
title: zsh 前台程序挂起
tags:
    - zsh
categories: Shell
excerpt: zsh 中使前台程序挂起及恢复
date: 2023-08-06 18:48:18
---

## 挂起

-   在 zsh 中，用户可以使用快捷键 `Ctrl + z` 挂起正在该终端中运行的前台任务。

## 查看后台任务

-   使用 `jobs` 查看当前 zsh 中被挂起的任务。

## 恢复

-   使用 `fg %<job_id>` 将挂起的任务重新恢复到前台。
-   若不加 `%<job_id>` 则会恢复最近挂起的任务。

## 后台恢复

-   使用 `bg %<job_id>` 将挂起的任务在后台恢复。
-   若不加 `%<job_id>` 则会恢复最近挂起的任务。

## 挂起后台任务

-   使用 `kill -STOP %<job_id>` 来挂起正在后台运行的任务。

## 使用相同快捷键恢复任务至前台

-   脚本来源： [sheerun](https://sheerun.net/2014/03/21/how-to-boost-your-vim-productivity/) 中的 VII. Use Ctrl-Z to switch back to Vim

```bash
fancy-ctrl-z () {
  if [[ $#BUFFER -eq 0 ]]; then
    BUFFER="fg"
    zle accept-line
  else
    zle push-input
    zle clear-screen
  fi
}
zle -N fancy-ctrl-z
bindkey '^Z' fancy-ctrl-z
```

{% folding black::ChatGPT 对于该脚本的解释 %}

This script defines a custom zsh widget (function) called `fancy-ctrl-z` and binds it to the Ctrl+Z key combination (`^Z`). It adds a bit of custom behavior to the default Ctrl+Z key binding, which is typically used to suspend the currently running foreground process.

Here's a breakdown of the script and its purpose:

1. `fancy-ctrl-z () { ... }`: This defines a shell function named `fancy-ctrl-z`. The function is intended to modify the behavior of the Ctrl+Z key combination.

2. `if [[ $#BUFFER -eq 0 ]]; then ... else ... fi`: This conditional statement checks whether the current input buffer (`$BUFFER`) is empty or not. In the context of this script, the input buffer likely represents the currently typed command line.

    - If the input buffer is empty, it means the user pressed Ctrl+Z without typing any command. In this case, the function sets the input buffer to "fg" (which is a common shell command to bring the most recently suspended process back to the foreground), and then it accepts the line (executes the command) using `zle accept-line`. Essentially, it behaves like the default behavior of Ctrl+Z, but with the "fg" command pre-filled.
    - If the input buffer is not empty (i.e., there is a command already typed), the function performs a different action. It saves the current input (using `zle push-input`), clears the screen (using `zle clear-screen`), and effectively resets the command line. This is a custom behavior where pressing Ctrl+Z when a command is already typed will clear the command line without executing the command.

3. `zle -N fancy-ctrl-z`: This tells zsh to create a new widget named `fancy-ctrl-z` using the function defined earlier.

4. `bindkey '^Z' fancy-ctrl-z`: This binds the `fancy-ctrl-z` widget to the Ctrl+Z key combination. Whenever you press Ctrl+Z, the custom function's behavior will be triggered.

In summary, this script customizes the behavior of Ctrl+Z in the zsh shell. If you press Ctrl+Z without typing a command, it behaves like the default Ctrl+Z by bringing the most recently suspended process back to the foreground. If you press Ctrl+Z with a command already typed, it clears the command line without executing the command. This can be useful for quickly clearing the command line when you change your mind about running a command or want to start typing a new one.

{% endfolding %}

{% notel orange fa-triangle-exclamation **注意** %}
需要排查快捷键冲突。
例如：`bindkey -e`
{% endnotel %}
