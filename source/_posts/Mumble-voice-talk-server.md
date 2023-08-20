---
title: Mumble 语音聊天服务器搭建
date: 2023-08-13 14:14:01
categories: 工具使用
tags:
    - server
    - ssh
    - docker
excerpt: 记录一次 mumble 语音聊天服务端的搭建过程
---

## 前言

-   在国内，对于语音聊天的选择并不多。由于墙的存在，最好用的 discord 被排除在了大部分人的选择范围之外，即使那些有能力使用的用户也无法兼顾代理和游戏加速器。
-   国内的主流语音聊天软件基本就是以下几个：
    -   微信/QQ：通话质量差，无法调节麦克风音量。
    -   yy：广告骑脸，系统资源占用高。
    -   kook：吃相难看，许多基础的功能都要收费。全面山寨 discord，体验却十分糟糕。

---

-   最近在看到一位 up 主发视频喷 kook 语音致使其 kook 帐号被封的事件后，决定此后再也不使用 kook。
-   在其视频的评论区看到了 teamspeak 这个替代品，然后顺藤摸瓜的找到了 mumble 这个开源的语音聊天软件。
-   本人并非什么极客，也没有什么开源洁癖，不过相较于闭源软件还是更喜欢开源，所以选择了 mumble。

## 阿里云服务器租用

-   我是学生，所以阿里云给我白嫖了一个月的 2 核 2G 内存 1 Mbps 带宽的 ecs 服务器，还是不错的。
-   后续似乎完成一个实验之后还可以再领取六个月的时间，甚好甚好。

### 开放 mumble 默认端口

-   在阿里云服务器控制台的安全组设置里开放 mumble 需要使用的端口：
    -   64738/tcp
    -   64738/udp

### ssh 远程连接

-   在阿里云服务器控制台重置 root 账户密码。
-   在本地目录 `~/.ssh` 下创建 `config` 文件：

```plaintext
Host murmur
    HostName <服务器公网 ip 地址>
    User root
```

{% notel orange fa-triangle-exclamation **注意** %}
ssh 默认使用 22 端口，如果远程服务器未开启此端口则无法连接。
{% endnotel %}

-   创建该文件后，即可使用 `ssh murmur` 命令访问远程服务器。

## Docker

-   趁着这次搭建 mumble 服务端的机会，学习一下 docker 的基础使用方法。

{% notel blue fa-circle-exclamation **注意** %}
下方命令均以 root 身份执行。
{% endnotel %}

### 安装 Docker

-   我选用的服务器系统是 Debian 11.7 64位，默认的 apt 源里没有 docker 。

{% notel orange fa-triangle-exclamation **注意** %}
根据[官方文档](https://docs.docker.com/engine/install/debian/)，docker engine 需要 64 位的 Debian 11/12，且系统内核版本必须大于 3.8。
{% endnotel %}

#### 使用官方安装脚本

-   参考[这篇博文](https://www.cnblogs.com/MicroTeam/p/see-docker-run-in-debian-with-aliyun-ecs.html)，命令如下：

```bash
apt install curl

curl -sSL https://get.docker.com/ | sh

service docker start
```

-   该博文中还提到阿里云将所有内网 IP 占用导致 docker 无法启动的情况，本人在操作时并未遇到，故不作记录。

#### 添加 apt 软件源（笔者采用）

-   参考[这篇博文](https://blog.csdn.net/qq_29753285/article/details/95094788)，命令如下：

```bash
# 必要工具
apt install apt-transport-https ca-certificates curl software-properties-common
# GPG 证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/debian/gpg | sudo apt-key add -
# 添加软件源
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/debian $(lsb_release -cs) stable"
# 刷新软件源
apt update
# 安装 docker
apt install docker-ce
# 启动 docker
service docker start
```

### 安装 Docker Compose

-   通过添加软件源的方式安装完 docker-ce(docker community edition) 后，compose 就已经被安装好了。
-   如果需要分别安装，添加的软件源内有 `docker-compose-plugin` 可供安装。

## murmur

### 镜像

-   [DockerHub](https://hub.docker.com/r/goofball222/murmur/#!) 上已经有了 murmur（mumble 服务端）的容器镜像，执行 `docker pull goofball222/murmur` 即可拉取镜像。

### 启动

-   使用命令行：

```bash
$ docker run --name murmur -d \
    -p 64738:64738/udp -p 64738:64738 \
    goofball222/murmur
```

-   使用 `docker compose` （推荐）：

{% folding gray::example docker-compose.yml %}

Example `docker-compose.yml` File：

```yml
version: "3"

services:
    murmur:
        image: goofball222/murmur
        container_name: murmur
        ports:
            - 64738:64738
            - 64738:64738/udp
        volumes:
            - /etc/localtime:/etc/localtime:ro
            - ./cert:/opt/murmur/cert
            - ./config:/opt/murmur/config
            - ./data:/opt/murmur/data
            - ./log:/opt/murmur/log
        environment:
            - TZ=UTC
```

{% endfolding %}

在 `docker-compose.yml` 所在的目录中执行：

```bash
docker compose up -d
```

### 配置

#### 设置超级用户密码

-   进入 docker 环境：

```bash
docker exec -it murmur bash
```

-   设置密码：

```bash
./murmur.x86 -supw <password>
```

#### 设置服务器密码

-   可以通过修改配置文件的方式来使服务器需要密码加入。
-   修改 `murmur.ini`：

```ini
; Password to join Server
serverpassword=<password>
```

-   保存后运行 `docker restart murmur` 来使之生效。

### 将用户设置为管理员

-   右键根频道（Root） -> 选择编辑（Edit） -> 选择用户组别（Group） -> 选择管理员（admin） -> 在下方的成员中添加用户。

{% notel blue fa-circle-exclamation **注意** %}
只有已注册的用户才能被设为管理员。
{% endnotel %}

### 证书 & 多设备登录

-   经测试，只要拥有相同的证书，就可以免去密码直接以创建证书的账户身份登录 mumble。
