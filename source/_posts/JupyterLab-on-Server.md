---
title: 服务器上可本地访问的 JupyterLab
date: 2023-11-05 20:54:33
categories: 工具使用
tags:
    - server
    - jupyter-lab
excerpt: 在服务器上配置可被本地访问的 JupyterLab
---

## 安装

```bash
pip3 install jupyterlab
```

-   或者：

```bash
micromamba install jupyterlab -c conda-forge
```

## 启动

```bash
jupyter-lab --no-browser --port 8889
```

## 本地远程访问配置

### 获取密钥

-   打开与 JupyterLab 同处一个虚拟环境的 python 交互式界面，执行以下命令：

```bash
   from jupyter_server.auth import passwd
   passwd()
```

-   依照提示输入两遍密码，将会得到一串密钥：`sha1:...`

### 生成配置文件

```bash
jupyter-lab --generate-config
```

### 修改配置文件

-   修改 `~/.jupyter/jupyter_lab_config.py` 中的以下内容：

```plaintext
c.ServerApp.allow_remote_access = True
c.ServerApp.ip = '*'

# open without browser
c.ServerApp.open_browser = False
c.LabServerApp.open_browser = False
c.ExtensionApp.open_browser = False
c.LabApp.open_browser = False

# the token just generated
c.ServerApp.password = 'sha1:...'

# port number
c.ServerApp.port = 8889
```
