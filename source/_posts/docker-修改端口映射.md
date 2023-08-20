---
title: docker 修改端口映射
date: 2023-08-20 12:25:26
categories: 工具使用
tags:
    - docker
excerpt: 如何修改 docker 容器的端口映射规则
thumbnail: https://s2.loli.net/2023/08/20/2pxXf3AbnPWM497.webp
---

## 前言

-   在 docker 启动某个容器时候，可以使用 `-p xxxx:yyyy` flag 来指定宿主机和容器内的端口映射。其中，xxxx 是宿主机端口，yyyy 是容器内端口。

-   有时会出现需要修改已经启动的容器的端口映射规则的情况。经查阅，大体有三种方法：

1. 修改 docker 配置文件
2. 根据该容器创建行的镜像及容器
3. 修改 iptables 端口映射

## 修改端口映射

### 修改 docker 配置文件

-   该方法需要重启整个 docker 服务。

1. 查看需要修改的容器 id：

```bash
docker ps -a
```

2. 关闭 docker 服务：

```bash
docker stop <container-id>
systemctl stop docker
```

3. 修改 `/var/lib/docker/containers/<container-id>/hostconfig.json`
   修改 `PortBindings`，例如：

```json
"PortBindings":{"8080/tcp":[{"HostIp":"","HostPort":"8091"}]}
```

该配置表明将宿主机的 8091 端口映射到了该容器内的 8080 端口。

4. 修改 `/var/lib/docker/containers/<container-id>/config.v2.json`
   修改 `ExposedPorts`，例如：

```json
ExposedPorts":{"8080/tcp":{}}
```

该配置表明容器内的 8080 端口对外开放。

5. 重启 docker 服务：

```bash
systemctl start docker
docker start <container-id>
```

### 新建容器

1. 提交一个运行中的容器作为镜像：

```bash
docker commit <container-id> <image-name>
```

2. 运行新建的镜像并建立新的端口映射：

```bash
docker run -d -p 8091:8080 <image-name>
```

### 修改 iptables 端口映射

-   略。
