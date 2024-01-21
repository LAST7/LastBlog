---
title: Jupyter Lab in Docker
date: 2023-11-14 11:26:23
categories: 工具使用
tags:
    - jupyter-lab
    - server
    - docker
    - python
    - micromamba
excerpt: 在服务器上部署运行在 Docker 中的 Jupyter Lab
---

## 前言

-   虽然直接在 NAS 服务器新建虚拟环境安装 JupyterLab 就完事了，但是由于之前使用 ansible 创建的应用全都被很好的包裹在容器内，令我感受到了一种 _前所未有的优雅_ 于是我决定将 Jupyter Lab 也使用容器运行，顺便学习一下 Dockerfile 的写法和基本的使用方法。
-   简而言之，生命在于折腾。

## Dockerfile

### 基础镜像

-   本人最终的目的在于创建一个为 PC 提供 Jupyter Lab 服务的容器，所以 Jupyter Lab 本身必不可少，与之相伴的 python 虚拟环境自然是不必多说。
-   由于本人相较于 conda 更喜欢 micromamba，所以此处也是如此选择。

-   根据[这个网站](https://micromamba-docker.readthedocs.io/en/latest/index.html)，官方给出了使用 Docker 配置的方法。
-   重点在于基础镜像 `mambaorg/micromamba`，有了这个镜像，就可以在 Dockerfile 里编写此后的配置/安装命令。

```Dockerfile
FROM mambaorg/micromamba
```

### 安装 Jupyter Lab 等库

-   在 Dockerfile 中写入：

```Dockerfile
RUN micromamba install -y -n base -c conda-forge jupyterlab

RUN micromamba install -y -n base -c conda-forge numpy scikit-learn matplotlib seaborn

RUN micromamba clean --all -y
```

{% notel orange fa-triangle-exclamation **注意** %}
注意：不推荐使用/创建 `base` 以外的环境。如果有需要，参照[此指示](https://micromamba-docker.readthedocs.io/en/latest/advanced_usage.html#multiple-environments)。
{% endnotel %}

### 设定工作目录 && 暴露端口

-   在 Dockerfile 中写入：

```Dockerfile
WORKDIR /JupyterCoding

# use port 8889 instead of 8888 to avoid collision
EXPOSE 8889
```

### 设置远程访问

-   参考[这篇文章](https://blog.imlast.top/2023/11/05/JupyterLab-on-Server/)，可将此前生成过的配置文件直接复制到相应的位置覆盖默认配置。
-   在 Dockerfile 中写入：

```Dockerfile
CMD ["jupyter", "lab", "--generate-config" "-y"]

# Place the previously generated config file in the same dir as Dockerfile
COPY --chown=$MAMBA_USER:$MAMBA_USER ./jupyter_lab_config.py /home/$MAMBA_USER/.jupyter/jupyter_lab_config.py
```

---

-   也可以设置一个 SSH 隧道，参考[这篇文档](https://docs.anaconda.com/free/anaconda/jupyter-notebooks/remote-jupyter-notebook/)。
-   在 PC 上执行：

```bash
ssh -L <PORT>:localhost:8889 <REMOTE_USER>@<REMOTE_HOST>
```

-   `<PORT>` 决定了本地 PC 上在浏览器中输入哪个端口。

{% notel orange fa-triangle-exclamation **注意** %}
该方法未经本人实践，请自行验证。
{% endnotel %}

### JupyterLab，启动！

```Dockerfile
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--no-browser", "--allow-root", "--port=8889"]
```

## 构建容器

-   有了 Dockerfile 后，构建容器就很简单了。只需在其目录中执行：

```bash
docker build -t <container-name> .
```

-   其中 `<container-name>` 是容器的名称。
-   该命令会自动在当前目录中寻找 Dockerfile，并用其构建容器。

## 容器，启动！

-   执行：

```bash
docker run -d -p 8889:8889 --restart=unless-stopped <container-name>:latest
```

-   其中 `-d` 表示 "detached"，`-p` 设定端口映射。
-   `--restart=unless-stopped` 意为除非手动停止运行，容器会自动重启。该选项可以达到使容器开机自启的效果。

## 坎坷

### 编辑/传入配置文件

-   第一次启动容器后，我本来想使用 `docker exec -it <id> /bin/bash` 的方式进入容器内部使用编辑器完成 `jupyter_lab_config.py` 的修改，结果发现基础镜像并没有安装编辑器……不论是 vim, vi 还是 nano 通通没有……
-   由于没有 root 权限，无法使用 apt 安装编辑器，所以从容器内部修改配置文件内容的方式在我看来是不可能的。
-   仔细查看了官网提供的 [Quick Start](https://micromamba-docker.readthedocs.io/en/latest/quick_start.html) 页面后，发现可以在 Dockerfile 中写入 `COPY` 命令完成文件的传入。
-   此外，询问大模型后得知了 `docker cp` 命令，可以向运行中的容器中传入文件。
