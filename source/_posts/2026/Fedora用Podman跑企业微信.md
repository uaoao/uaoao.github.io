---
title: Fedora用Podman跑企业微信
date: 2026-07-26
tags:
  - 年份-2026
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-Linux
  - 主题-Podman
  - 主题-KVM
  - 主题-企业微信
  - 主题-Fedora
---

参考 [通用技术沉淀 | 第0篇：Linux下轻量化运行企业微信](https://arthaks.github.io/posts/linux-wecom-docker-rdp/)  这篇文章。

原文用的是Ubuntu，如果跑在Fedora有点问题，SELinux会阻止容器访问宿主机绑定的文件系统。以下操作与原文大致相同。

## 下载 Tiny10 镜像

如果你网络有条件，可以跳过这一步，否则请提前下载好镜像。并在跑容器时指定镜像文件。

- [Tiny10 23H2](https://archive.org/details/tiny-10-23-h2)

## 拉取镜像，跑容器

容器下载很快。容器跑起来后自动下载Windows镜像贼慢。下面的命令使用了本地镜像加载。

```bash
podman run \
    --name WeCom \
    --restart unless-stopped \
    --cap-add NET_ADMIN \
    --device /dev/kvm \
    --device /dev/net/tun \
    -e RAM_SIZE="2G" \
    -e CPU_CORES="2" \
    -e DISK_SIZE="64G" \
    -e USERNAME="wecom" \
    -e PASSWORD="wecom" \
    -p 8006:8006 \
    -p 3391:3389/tcp \
    -p 3391:3389/udp \
    -v $PWD/tiny10_x64_23h2.iso:/custom.iso:z \
    -v wecom_data:/storage \
    -v $HOME/Documents/WeCom:/shared:z \
    -d ghcr.io/dockur/windows:latest

```

执行 `podman logs -f WeCom` 查看后台日志；点击 <http://localhost:8006/> 进入虚拟机桌面，与主机共享的文件夹在桌面，名为【Shared】。你应该注意到了，这个容器其实只是一个安装和控制程序，底层调用的是KVM。

## 设置系统

- 设置中文：打开【Settings】，找到【Time&Language】，点击【Language】，在【Preferred language】一栏点击【Add a language】，搜索`Chinese`，勾选安装并设置为系统首选语言。
- 关闭窗口动画：打开【Settings】，搜索【adjust the appearance】，在视觉效果选项卡选择最佳性能并保存。
- 快捷栏显示浏览器：点击左下角Win徽标，对Edge浏览器右键选择【更多——固定到任务栏】
- 下载安装输入法，比如微信输入法。
- 下载安装企业微信，不要勾选开机启动。

## 结语

开机登录主机后自动启动容器，想暂时停止运行，在终端执行 `podman stop WeCom` 将容器关机。由于我使用的是Fedora Sliverblue，原文后续操作未能在我电脑上实现，基本只是用浏览器访问桌面。这种方案有以下几个缺点：

1. KVM运行操作系统，不够轻量省电。
2. 宿主机与Win互通是通过【Shared】文件夹，这个文件夹是快捷方式，指向了一个主机应用程序无法访问的网络位置，这就导致企业微信无法配置直接保存文件到共享文件夹中，必须手动将文件下载移动到【Shared】文件夹内。
3. 浏览器操作，不够本地化。
4. 挂载USB外设不方便，必须重新创建容器时修改设备ID。但是我测试发现不管U盘还是光驱都无法挂载进容器，日志没输出报错但是容器内没有这个设备。

其中第二点和第四点是日常工作最频繁使用的场景。折腾半天结果不尽人意，遂放弃此方案。

## 相关参考链接

- [Windows inside a Docker container](https://github.com/dockur/windows)
