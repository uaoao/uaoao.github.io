---
title: 在OpenWrt的Docker上部署Beszel监控
date: 2025-6-15
tags:
  - 年份-2025
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-技术知识
  - 主题-OpenWrt
  - 主题-Linux
  - 主题-Docker
  - 主题-Beszel
  - 主题-建站
---

[Beszel](https://beszel.dev/zh/) 是一个轻量级的分布式系统监控工具，适合一些简单的系统资源监控任务。它分为 **HUB** 和 **Agent**两个部分。**Agent** 部署在小规模的各种机器上收集信息，**HUB** 则负责将各个 **Agent** 收集的信息整理到一个数据库【PocketBase】中，并在网页端展示。

我的 OpenWrt 路由器部署了一些像 Nextcloud 这种吃资源的服务，需要一个监控工具来查看资源占用情况。感觉 Beszel 挺符合需求，于是在这里分享一下如何将HUB和Agent部署在同一台机器上。

## 前提条件

1. OpenWrt 安装了 Docker
2. 使用 `docker network create --subnet 172.20.20.0/24 --gateway 172.20.20.1 br-2020` 创建自定义接口 `br-2020`。
3. 反向代理，这里以 Cloudflare 为例子

## 安装和部署

1. SSH 登陆 OpenWrt，先安装 HUB 端。其中 `CSP` 变量的作用是允许 Beszel 嵌入在其他网站的框架内。~~我希望利用 Nextcloud 的【[External sites](https://apps.nextcloud.com/apps/external)】插件实现嵌入访问。[来源](https://github.com/henrygd/beszel/issues/112#issuecomment-2381603289)~~

```bash
#!/bin/sh

docker run \
  --name beszel \
  --restart=unless-stopped \
  --network br-2020 \
  --ip 172.20.20.3 \
  -v /root/beszel/data:/beszel_data \
  -v /root/beszel/socket:/beszel_socket \
  -e TZ=Asia/Shanghai \
  -e SKIP_SYSTEMD=true \
  -e SKIP_GPU=true \
  -d henrygd/beszel:latest

# 下面这行被我移除了，如果你需要可以加上
#  -e CSP="frame-ancestors https://nextcloud.example.com https://nctun.example.com" \

```

2. 完成后，打开浏览器。在 Cloudflare Zero Trust Tunnels 中添加一条 Tunnel，绑定域名 `monitor.example.com`。我这里就不多解释了，可以看【[OpenWrt配置Cloudflare隧道](/2025/5/27/OpenWrt%E9%85%8D%E7%BD%AECloudflare%E9%9A%A7%E9%81%93.html)】这篇文章。

> 注意：由于后台数据库【PocketBase】的限制，用 Cloudflare 做反向代理必须在该隧道的 **HTTP设置** 中启用 **禁用分块编码**。否则注册账号后会被强制退出无法正常登陆。[来源](https://github.com/pocketbase/pocketbase/discussions/6663)

3. 打开 `monitor.example.com`，注册账号并进入面板后点击右上角【Add System】添加服务器，填入【Host/IP】，点击复制公钥和令牌。

【Name】`OpenWrt`
【Host/IP】`/beszel_socket/beszel.sock`
【Public Key】`ssh-ed25519 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
【Token】`xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

4. SSH 登陆 OpenWrt，然后安装 Agent 端。

```bash
#!/bin/sh

docker run \
  --name beszel-agent \
  --restart=unless-stopped \
  --network host \
  -v /root/beszel/agent:/var/lib/beszel-agent \
  -v /root/beszel/socket:/beszel_socket \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -e TZ=Asia/Shanghai \
  -e LISTEN=/beszel_socket/beszel.sock \
  -e HUB_URL="https://monitor.example.com" \
  -e TOKEN="xxxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
  -e KEY="ssh-ed25519 xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  -d henrygd/beszel-agent:latest

```

5. 回到 `monitor.example.com`，点击添加，就能正常看到 OpenWrt 的系统资源信息了。

## 相关参考链接

- [Beszel Github](https://github.com/henrygd/beszel)

