---
title: OpenWrt配置使用Docker
date: 2025-04-18
tags:
  - Docker
  - OpenWrt
  - Linux
  - 配置
  - 2025年
---

## 安装

安装全套软件包，建议不要在网页上安装，可能会有一些奇怪的问题。使用以下命令安装全套Docker软件包：

```bash
opkg update
opkg install luci-app-dockerman
reboot
```

## 配置镜像服务器

由于众所周知的原因，某些地区无法直接访问Docker Hub，所以需要配置镜像服务器。而曾经某些高校和企业提供的镜像服务器也由于某些不可抗力因素停止服务。我这里提供几个截至2025年4月可用的镜像服务器配置和可用的订阅，不保证永久可用：

- 在`/etc/config/dockerd`中配置镜像服务器（或者网页 Docker -> Configuration 配置）：

```txt
	list registry_mirrors 'https://dytt.online'
	list registry_mirrors 'https://lispy.org'
	list registry_mirrors 'https://docker.xiaogenban1993.com'
	list registry_mirrors 'https://docker.yomansunter.com'
	list registry_mirrors 'https://aicarbon.xyz'
	list registry_mirrors 'https://666860.xyz'
	list registry_mirrors 'https://docker.zhai.cm'
	list registry_mirrors 'https://a.ussh.net'
	list registry_mirrors 'https://docker.1ms.run'
	list registry_mirrors 'https://docker.mybacc.com'
```

- 订阅网站：

1. https://github.com/dongyubin/DockerHub
2. https://github.com/cmliu/CF-Workers-docker.io
