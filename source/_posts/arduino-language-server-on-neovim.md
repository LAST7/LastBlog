---
title: 在 Neovim(v0.10+) 上使用 arduino-language-server
date: 2025-02-22 11:05:34
categories: 工具使用
tags:
    - embedded development
    - arduino
    - neovim
    - LSP
excerpt: 在 Neovim 中安装并使用 arduino-language-server 时所遇到的问题和解决方案
---

## 前言

- 毕设选了个嵌入式开发的小项目，不是很想用笨重的 arduino IDE。
- 于是浪费了半个下午配置环境……水篇博客，不然这努力岂不是白费了（bushi

## 正文

### 问题描述

- 期初我尝试直接使用 `Mason` 安装 `arduino-language-server`，但即便按文档要求修改配置后仍然无法正常启动。
- 查阅相关 [issues/PR](https://github.com/arduino/arduino-language-server/pull/199#issuecomment-2453517587) 之后发现，问题不在 `arduino-language-server` 自身，而是将其 integrate 进 Neovim 时所依赖的 `go-lsp` 上。

    > The dependency (bugst/go-lsp) used by the Arduino language server was updated (its go.mod was bumped to Go 1.22), and its behavior causes the language server to “fail to attach” or “quit” when Neovim v0.10+ attempts to start it. This manifests as errors in the Neovim logs (e.g. errors related to clangd startup and “locked (waiting clangd)”, even though the server may still work despite the warning message).

- [speelbarrow](https://github.com/speelbarrow) 将 `arduino-language-server` 依赖的 `go-lsp` 更换为了其 patch 过的版本：

    ```diff
    +++ go.mod
    @@ 32,3 +32,5 @@
        gopkg.in/check.v1 v1.0.0-20201130134442-10cb98267c6c // indirect
        gopkg.in/yaml.v3 v3.0.1 // indirect
    )
    +
    + replace go.bug.st/lsp => github.com/speelbarrow/go-lsp v0.1.3-0.20241103164431-cf1c00fb5806
    ```

- 进一步查看其对于 `go-lsp` 所做的修改：

    ```diff
    --- jsonrpc/jsonrpc_connection.go
    +++ jsonrpc/jsonrpc_connection.go
    @@ -277,7 +277,7 @@ func (c *Connection) SendRequest(ctx context.Context, method string, params json
            _, active := c.activeOutRequests[id]
            c.activeOutRequestsMutex.Unlock()
            if active {
    -           if notif, err := json.Marshal(CancelParams{ID: encodedId}); err != nil {
    +           if notif, err := json.Marshal(CancelParams{ID: encodedID}); err != nil {
                           // should never happen
                           panic("internal error: failed json encoding")
                      } else {
    ```

- 原来只是一个类型 typo 错误吗……

### 解决方案

- 虽然上文提到的 patch 已经能够使得 `arduino-language-server` 运行在 neovim v0.10+ 的版本中，但是该 PR 并未被 merge。查看 `go-lsp` 仓库的 Insights 界面可知：

    ![screenshot_22022025_114218.jpg](https://s2.loli.net/2025/02/22/1jNncSWpA6PZkLv.png)

- ~很显然这个仓库已经死了。~

- 仔细看了一下，这个仓库似乎仅仅是为了 `arduino-language-server` 而 fork 的一个仓库，作者 [cmaglie](https://github.com/cmaglie) 已经很久没有对其进行维护了。

---

- 尽管如此，办法也还是有的：

    - 将 [speelbarrow 的 repo](https://github.com/speelbarrow/arduino-language-server/tree/main) 克隆到本地：

        ```bash
        git clone https://github.com/speelbarrow/arduino-language-server.git
        ```

    - 手动 build 一个依赖正确 `go-lsp` 的 `arduino-language-server`：

        ```bash
        cd arduino-language-server && go build
        ```

    - 将 Mason 里安装的可执行文件替换为手动构建的 `arduino-language-server`：

        ```bash
        cp ./arduino-language-server ~/.local/share/nvim/mason/packages/arduino-language-server/arduino-language-server
        ```

## 配置文件

- 顺带的，`arduino-language-server` 的启动需要一些前置步骤以及一些命令行参数。

- 前置步骤详见： [lspconfig doc](https://github.com/neovim/nvim-lspconfig/blob/master/doc/configs.md#arduino_language_server)
- 本人开发 ESP32 CAM 所需要的命令行参数如下：

    _astrolsp.lua:_

    ```lua
    arduino_language_server = {
        on_new_config = function(config)
            config.capabilities.textDocument.semanticTokens = vim.NIL
            config.capabilities.workspace.semanticTokens = vim.NIL
            config.cmd = {
                "arduino-language-server",
                "-clangd",
                "/home/last/.local/share/nvim/mason/bin/clangd",
                "-cli",
                "/usr/bin/arduino-cli",
                "-cli-config",
                "/home/last/.arduino15/arduino-cli.yaml",
                "-fqbn",
                "esp32:esp32:esp32cam",
            }
        end,
    },
    ```

    _注意：该配置仅适用于 AstroNvim 下，且需要使用 Mason 安装 `clangd`_

## 总结

- 又是被环境配置折磨的一天。
- 这麻烦还是我自己找的，哈哈。
