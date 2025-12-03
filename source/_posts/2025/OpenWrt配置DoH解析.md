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

OpenWrt 对DoH的支持相较于DoT更好一点。具体表现在其简易的配置方式：我们不需要使用命令行去修改特定的哪个文件，只需安装对应的程序即可默认启用DoH。

当然，作为盐碱地用户，直接用默认配置是无法正常上网滴，必须修改成能正常解析的DoH服务商，这个步骤就非常考验咱们的耐心。能正常使用的情况下，使用海外DoH是最优选择。当海外DoH无法解析时，就使用盐碱地的DoH兜底解析。有些人可能受不了海外解析的延迟，那么只能用盐碱地的DoH解析了。我是有着宁可忍受超高延迟也尽可能不用盐碱地DoH的决心。毕竟，保障人身安全是第一位！而且海外DoH的另一大优势就是能解析一些被污染的域名（被屏蔽IP的服务器除外）。

## 配置过程

1. 安装 `luci-app-https-dns-proxy` 这个软件包。

2. 重新登陆OpenWrt，进入 Services -> HTTPS DNS Proxy。

安装好后服务默认就处于Enable和Running状态，这时候身处盐碱地的你会发现突然无法上网了！！！不仅不能访问某些被污染的站点，连盐碱地的网站也不能访问了。

3. 安装好后默认会使用Google和Cloudflare的DoH服务器。点击Edit，修改以下内容：

- 【Provider】修改为盐碱地的DNS服务器，比如 `Ali DNS`
- 【Bootstrap DNS】修改为盐碱地的DNS服务器，比如 Ali DNS `223.6.6.6,2400:3200:baba::1,223.5.5.5,2400:3200::1`
- 【Listen Address】不填
- 【Listen Port】保持默认 `5054` （不能填入已占用的端口）
- 【Run As User】不填
- 【Run As Group】不填
- 【Proxy Server】如果使用盐碱地之外的DoH服务器，并且想让DoH通过代理加速，可以填写你的代理服务器监听地址
- 【Use HTTP/1】保持默认
- 【Use IPv6 resolvers】保持默认

提示：【Bootstrap DNS】是指解析 DoH 服务器的 DNS 服务器。因为 DoH 走 HTTPS 协议需要先获取其服务器的 IP 地址才能正常使用，建议填盐碱地的 DNS 服务器 IP 地址。

4. 最后保存，重启服务就可以正常使用了。

## 测试哪些 DoH 在你所处地区可用

在 [这里](https://github.com/stangri/luci-app-https-dns-proxy/tree/main/root/usr/share/https-dns-proxy/providers) 可以找到所有能配置的 DoH 供应商。选择一个供应商，找到其中的 `template` 键值对（如果值包含 `{option}` 需要用 `params.option.options.value` 的值替换），复制。

然后在命令行中执行以下命令（以 `https://doh.360.cn/dns-query` 为例）：

```bash
DNS_URL=https://doh.360.cn/dns-query
curl -H 'accept: application/dns-message' "${DNS_URL}?dns=q80BAAABAAAAAAAAA3d3dwdleGFtcGxlA2NvbQAAAQAB" | hexdump -c

```

查询结果是 `www.example.com` 的 IP地址。不管查询内容如何，能获得结果就表明可正常使用。如果你想使用 Cloudflare 提供的私人 DoH 服务，配置时【Provider】选择【Custom】，在【Parameter】这一栏添加自己的服务器链接，其他的内容按前文配置。注意，你的运营商 DNS 可能会污染测试用的域名导致测试结果不准确，建议先配置一个可用的 DoH 后再测试。

## 内网设置了 DoT 的设备无法联网？

因为 HTTPS DNS Proxy 这个服务默认拦截 `53` 和 `853` 两个端口。内网配置了 DoT 的设备，比如设置了私人DNS服务的手机，必须用 `853` 端口解析 IP 地址，不会使用 OpenWrt 的配置。

需要在 OpenWrt 命令行中编辑 `/etc/config/https-dns-proxy` 注释掉 `list force_dns_port '853'` 这一行，然后执行 `/etc/init.d/https-dns-proxy restart` 重启服务。

## 相关参考链接

- [国内目前可用的DoH](https://coding.gs/2024/06/09/available-doh/)
- [DNS over HTTPS（DoH）协议分析](https://jia.je/kb/software/doh.html)

