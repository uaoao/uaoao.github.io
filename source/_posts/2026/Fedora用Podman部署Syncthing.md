---
title: Fedora用Podman部署Syncthing
date: 2026-10-17
tags:
  - 年份-2026
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-Syncthing
  - 主题-Fedora
  - 主题-Podman
  - 主题-SELinux
  - 主题-备份
---

以前写过一篇文章【[在OpenWrt的Docker上部署qBittorrent和Syncthing服务](/2025/12/10/在OpenWrt的Docker上部署qBittorrent和Syncthing服务.html)】，简单介绍了如何在 OpenWrt路由器操作系统上部署 Syncthing。这种服务只部署一个节点没有任何意义，至少需要两个，才能互相同步数据。比如在手机或者电脑上用 Syncthing。

在写这篇文章之前，我一直用手机app【Syncthing-fork】作为另一个节点连接，毕竟手机弄丢的可能性大。

最近笔记本换新，硬盘也换了1T固态，数据迁移的时候有一部分工作数据忘记迁移，连同旧硬盘被一同格式化灰飞烟灭。幸亏在另一台 Windows 笔记本以及服务器上有零星分散的文件，不至于造成重大数据事故，及时恢复并重建了工作流。

考虑到之后可能还会发生类似情况，我决定在笔记本上解决Syncthing同步的问题。

刚学会用 Syncthing 部署在 OpenWrt 时就尝试过在 Fedora 笔记本上部署这个服务。当时由于 SELinux 和 UID/GID 的问题一直没解决，用 `dnf install syncthing` 安装的服务倒不会出现权限问题，可是我有强烈的系统洁癖，能不用系统 `dnf` 装用户APP就绝对避免使用。尤其作为一名 Silverblue 用户，容器化是保证系统干净整洁的绝对权威。

我先后尝试通过 Podman 安装、通过包管理器安装、通过 `toolbox` 创建的 Fedora 容器内安装，最后都没达到令我满意的完美解决方案（至于 Podman 这方面，则是因为权限问题一时没法解决）。

直到今天，Podman 部署容器的方式才被完美地解决。

## 部署 Syncthing

```bash
#!/bin/bash

podman run \
  --name syncthing \
  --restart unless-stopped \
  --network host \
  --security-opt label=disable \
  --tz "Asia/Shanghai" \
  -e PUID=0 \
  -e PGID=0 \
  -v syncthing-data:/config \
  -v $HOME/Sync:/root/Sync:z \
  -d lscr.io/linuxserver/syncthing:latest

```

## 选项解读

关键在于以下四个选项：

- `--security-opt label=disable` 用于设置容器的SELinux行为，这里是禁用了容器的SELinux，防止容器无法访问挂载了主机文件系统卷的文件。
- `-v $HOME/Sync:/root/Sync:z` 重新标记SELinux上下文，使得容器与容器、容器与主机之间的文件可以互相访问（但是仅容器启动时对现有文件生效，容器运行阶段不会自动重新标记新文件），大写的`Z`则是仅限当前容器与主机可互相访问。
- `-e PUID=0` 设定容器内的进程以root权限执行（Rootless容器映射到主机就是普通用户UID=1000）
- `-e PGID=0` 设定容器内的进程以root权限执行（Rootless容器映射到主机就是普通用户GID=1000）

另外，推荐使用 `--network host` 作为容器网络，可以获取到IPV6，节点之间就能直连无须中继器，上传下载速度就取决于自身和对方的带宽。

## 访问管理页面

访问 [http://localhost:8384/](http://localhost:8384/) 就是Syncthing管理页面。

注意密码一定要设置，防止别人通过IPV6扫到你的端口。更安全的方法是修改系统默认的防火墙Zone 从 `FedoraWorkstation` 到 `public`，这会屏蔽大于 `1025` 的所有端口，然后再放行 `22000/tcp` `22000/udp` `21027/udp` 这三个端口即可，本机可以正常通过 `8384` 端口访问管理页面。

## 相关参考链接

- [How to Fix SELinux Denials with Podman Containers](https://oneuptime.com/blog/post/2026-03-18-fix-selinux-denials-podman-containers/view)
- [linuxserver/syncthing](https://docs.linuxserver.io/images/docker-syncthing/)
