---
title: OpenWrt配置DoH解析
date: 2025-04-24
tags:
  - 年份-2025
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-技术知识
  - 主题-OpenWrt
  - 主题-Linux
  - 主题-DoH
---

## 前言

OpenWrt 对DoH的支持相较于DoT更好一点。具体表现在其简易的配置方式：我们不需要使用命令行去修改特定的哪个文件，只需安装对应的程序即可默认启用DoH。

当然，作为盐碱地用户，直接用默认配置是无法正常上网滴，必须修改成能正常解析的DoH服务商，这个步骤就非常考验咱们的耐心。能正常使用的情况下，使用海外DoH是最优选择。当海外DoH无法解析时，就使用盐碱地的DoH兜底解析。有些人可能受不了海外解析的延迟，那么只能用盐碱地的DoH解析了。我是有着宁可忍受超高延迟也尽可能不用盐碱地DoH的决心。毕竟，保障人身安全是第一位！而且海外DoH的另一大优势就是能解析一些被污染的域名（被屏蔽IP的服务器除外）。

## 配置过程

1. 安装 `luci-app-https-dns-proxy` 这个软件包。

2. 重新登陆OpenWrt，进入 Services -> HTTPS DNS Proxy。

安装好后服务默认就处于Enable和Running状态，这时候身处盐碱地的你会发现突然无法上网了！！！不仅不能访问某些被污染的站点，连盐碱地的网站也不能访问了。

3. 安装好后默认会使用Google和Cloudflare的DoH服务器。点击Edit，修改以下内容：

- Provider 修改为盐碱地的DNS服务器，比如 `Ali DNS`
- Bootstrap DNS 修改为盐碱地的DNS服务器，比如 Ali DNS `223.6.6.6, 2400:3200:baba::1, 223.5.5.5, 2400:3200::1`
- Listen Address 保持默认 `127.0.0.1`
- Listen Port 保持默认 `5054` （不能填入其他进程已占用的端口）
- Run As User 保持默认 `nobody`
- Run As Group 保持默认 `nogroup`
- Proxy Server 如果使用盐碱地之外的DoH服务器，并且想让DoH通过代理加速，可以填写你的代理服务器监听地址
- Use HTTP/1 保持默认 `Use negotiated HTTP version`
- Use IPv6 resolvers 保持默认 `Use any family DNS resolvers`

提示：Bootstrap DNS 是指解析 DoH 服务器的 DNS 服务器。因为 DoH 走 HTTPS 协议需要先获取其服务器的IP地址才能正常使用，建议填盐碱地的DNS服务器IP地址。填写 Proxy Server 需要代理本身不受OpenWrt的DNS影响（大概）

4. 最后保存并重启服务就可以正常使用了

## 盐碱地能用的 DoH Provider

以下提供了一些我家宽带可用的DoH服务商。你可以先只留一个DoH服务商，然后修改服务商逐个测试是否能正常加载网站。

### 海外 DoH

- Applied Privacy DNS(AT)
- CIRA Canadian Shield
- Comss DNS(RU)
- DeCloudUs DNS
- Digitale Gesellschaft(CH)
- DNS For Family
- DNS Forge(DE)
- Rethink DNS

...太多了暂时测了这些。由于有DNS缓存所以可能结果不一定准确

### 盐碱地 DoH

- Ali DNS
- DoH 360 DNS(CN)
