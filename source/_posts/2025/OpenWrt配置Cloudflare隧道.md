---
title: OpenWrt配置Cloudflare隧道
date: 2025-5-27
tags:
  - 年份-2025
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-技术知识
  - 主题-OpenWrt
  - 主题-Cloudflare
  - 主题-Linux
  - 主题-建站
---

Cloudflare Zero Trust 为免费用户提供了隧道功能，可以将内网设备一些特定的端口和服务暴露在公网环境中。方便直接的部署、无需配置防火墙、约等于内网穿透的互联网访问能力、IPv4和IPv6皆可访问，这些优势或许比DDNS服务绑定动态公网IPv6地址还要简单稳定。而且，本地主机到Cloudflare服务器之间的通信完全加密；若主机对外提供的是HTTP服务，Cloudflare还会自动为这个域名提供HTTPS认证服务，相当便捷。

唯一的缺点就是带宽受限以及在盐碱地访问网站丢包比较严重。

这篇文章以公网访问HTTPS的Openwrt登陆页面为例子说明配置方法。请在配置之前**设置16位以上的随机复杂root密码**，本文仅仅是以Openwrt登陆页面作为示例，其他内网主机提供的网页服务也同样适用，**不要将Openwrt登陆页面长期暴露在公网环境**。

## 准备

- 域名一个，且DNS解析托管在Cloudflare，比如 `example.com`
- Cloudflare开通了Zero Trust，绑定了上述域名
- Openwrt系统能够安装软件
- 手机或其他与Openwrt不在同一个局域网内的电脑

## 获取Token和证书文件

1. 登陆 [Cloudflare Zero Truet](https://one.dash.cloudflare.com/)
2. 点击左侧 Networks —— Tunnels，点击 Create a tunnel，选择隧道类型为 Cloudflared
3. 填写隧道名称，我这里就填写 gateway，然后点击保存隧道
4. 页面中有三块命令行文本框，只需复制底下两个中的一个就行，暂时用记事本保存
5. 点击网页底部的下一步
6. 在 Hostname 添加想对外提供访问的域名，比如 `gateway.example.com`；在 Service 添加服务协议类型和内网中的主机地址，比如 HTTP 和 `192.168.1.1`（只能由IP地址和端口号构成，若是IPv6地址，则需用中括号包括IP地址），然后保存
7. 打开 [Authorize Cloudflare Tunnel](https://dash.cloudflare.com/argotunnel)，点击上述域名，点击认证按钮，浏览器会下载一个 `cert.pem` 文件

## 配置 Openwrt 的 Cloudflare Zero Trust Tunnel

1. 登陆 Openwrt，在 System —— Software 中点击 Update lists
2. 查询软件包 `luci-app-cloudflared` 然后安装它
3. 注销重新登陆 Openwrt，打开 VPN -- Cloudflare Zero Trust Tunnel
4. 填写之前复制的 Token（删掉前面的命令部分，只填写随机字符串部分），在Certificate of Origin 中上传之前下载的 `cert.pem` 文件
5. 勾选 Enable，然后点击保存
6. 刷新页面后现实 Running，再用手机或其他与Openwrt不在同一个局域网内的电脑访问配置的网址，比如 `https://gateway.example.com`，如果能成功访问，则配置成功

至此，以公网访问HTTPS的Openwrt登陆页面的例子就完成配置了。

## 添加第二个网站

添加第二个网站不要重复前面的步骤，只需编辑已创建的隧道添加一个新主机即可。

1. 登陆 [Cloudflare Zero Truet](https://one.dash.cloudflare.com/)，在 Networks —— Tunnels 中点击先前配置好的隧道，点击编辑
2. 点击 Public Hostname 选项卡，点击 Add a public hostname
3. 在 Hostname 添加想对外提供访问的域名，比如 `files.example.com/share`(**注意，`share`这个路径在主机中必须存在且可访问**）；在 Service 添加服务协议类型和内网中的主机地址，比如 HTTP 和 `192.168.1.180:8888`，然后保存
4. 用手机或其他与Openwrt不在同一个局域网内的电脑访问配置的网址，比如 `https://files.example.com/share`，如果能成功访问，则配置成功

## 相关参考链接

- [CloudFlare Tunnel 免费内网穿透的简明教程](https://sspai.com/post/79278)
- [Cloudflare tunnel](https://openwrt.org/docs/guide-user/services/vpn/cloudfare_tunnel)
