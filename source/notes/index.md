---
title: 备忘录
type: notes
cover: img/notes.webp
---

- [Linux配置](Linux配置)
- [Bash脚本](Bash脚本)
- [OpenWrt配置](OpenWrt配置)
- [Android-App配置](Android-App配置)

# DNS

## AliDNS

```txt
https://dns.alidns.com/dns-query
223.6.6.6
223.5.5.5
2400:3200:baba::1
2400:3200::1

```

## DNS Over Https

- <https://dns-doh.dnsforfamily.com/dns-query>
- <https://xxx.ddd.oaifree.com/query-dns>
- <https://doh.mullvad.net/dns-query>

## Win10 跳过联网安装

在选择网络界面按【Shift-F10】打开终端，输入下面的指令创建本地用户账号。

```bash
start ms-cxh:localonly

```