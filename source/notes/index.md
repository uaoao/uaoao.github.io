---
title: 备忘录
type: notes
cover: img/notes.webp
---

- [Linux配置](Linux配置)
- [Bash脚本](Bash脚本)
- [OpenWrt配置](OpenWrt配置)
- [Android-App配置](Android-App配置)

## DNS 解析

阿里 DNS 服务只适用于国内网站的域名解析，或者用于解析 DoT/DoH DNS 的域名。大量无备案域名的IP很少更新，可能解析到无法访问的IP地址。

```txt
https://dns.alidns.com/dns-query
223.6.6.6
223.5.5.5
2400:3200:baba::1
2400:3200::1

```

可用的海外 DoH 地址

```txt
https://dns-doh.dnsforfamily.com/dns-query
https://xxx.ddd.oaifree.com/query-dns
https://doh.mullvad.net/dns-query

```

## Win10 跳过联网安装

在选择网络界面按【Shift-F10】打开终端，输入下面的指令创建本地用户账号。

```bash
start ms-cxh:localonly

```

## Mysql5.7 配置文件

配置文件在 `/etc/mysql/config.d/my.cnf`

```ini
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[client]
default-character-set = utf8mb4

[mysql]
default-character-set = utf8mb4

```

## 图例

- BIOS快捷键

![BIOS快捷键](/images/notes/BIOS快捷键.webp)

- Linux 常用命令

![Linux 常用命令](/images/notes/Linux常用命令.webp)

- Linux弃用的命令和替代品

![Linux弃用的命令和替代品](/images/notes/Linux弃用的命令和替代品.webp)

- Vim 键盘布局

![Vim 键盘布局](/images/notes/Vim键盘布局.webp)

- 数显测电笔使用说明

![数显测电笔使用说明](/images/notes/数显测电笔使用说明.webp)

- 中文编码错误

![中文编码错误](/images/notes/中文编码错误.webp)
