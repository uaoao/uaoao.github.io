---
title: 为Fedora43配置Cloudflare的私人DoT
date: 2025-11-6
tags:
  - 年份-2025
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-Linux
  - 主题-Fedora
  - 主题-Cloudflare
  - 主题-技术知识
  - 主题-DoT
  - 主题-翻译
  - 主题-网络安全
---

现代网站越加重视网络安全和连接加密，尤其是在ECH不断推广受浏览器和网页服务器支持的现在和未来，防范DNS和SNI被监控和污染阻断是实现自由安全上网的一大步。

最近学习了HTTPS连接建立的过程（早就该学了只是太懒一直拖着没研究），随后发现连接建立过程中Client Hello默认不加密，这就给别有用心之人留下了利用SNI监控用户和阻断连接的把柄。

早些年开始推广ESNI，也就是加密Client Hello中的域名。近三年发展到ECH加密Client Hello中所有信息，很快就替代了ESNI作为主流浏览器默认启用的功能。再搭配HSTS限制客户端仅HTTPS连接，加密通信才算完全。唯一的遗憾就是无法加密也不可能加密连接的IP地址和端口，只能代理绕过了。

想要使用ECH，DoH/DoT是关键。ECH就是靠加密DNS提供信息指向正确的网站。目前主流的操作系统，包括Linux发行版、Android、Windows、MacOS都已支持启用和DoT相关配置。本文主要介绍如何在Fedora上配置Cloudflare Zero Trust提供的免费私人DoT服务。这个服务的域名、IP和443、853端口目前在部分地区没有被墙，之前折腾OpenWrt时也用它来提供DoH。

> 以下内容大部分翻译自 [Configure DoT on systemd-resolved](https://shivering-isles.com/2021/01/configure-dot-on-systemd-resolved) 略有增删。

## Cloudflare Zero Trust添加默认DoT

1. 进入 [Zero Truest](one.dash.cloudflare.com) —— Gateway —— DNS位置。
2. 点击 **添加位置**。
3. 在 **选择DNS终结点** 处勾选 **基于TLS的DNS（DoT）**，其他三个选项可选，建议全部勾选。
4. 在 **默认DNS位置** 处勾选 **启用EDNS客户端子网**。如果这是你创建的唯一DNS配置，就把 **设置为默认DNS位置** 勾上。
5. 点击 **继续**，如果想进一步限制哪些IP地址可用这个DNS配置，就勾选 **DoT终结点筛选** 后添加对应的IP，否则就忽略。
6. 没问题就点击 **完成**。
7. 复制 **基于TLS的DNS** 中的域名，比如 `abc123.cloudflare-gateway.com`，接下来就使用这个域名为操作系统配置全局DoT。

## 配置 systemd-resolved

首先你可以使用下面这个小脚本创建和更新DNS配置。**注意把【abc123】换成你自己创建的真实子域名**。

```bash
#!/bin/sh

RESOLVER="${1:-abc123.cloudflare-gateway.com}"

DNS_SERVERS=""

for dns in $(dig ${RESOLVER} +short && dig ${RESOLVER} +short AAAA); do
    DNS_SERVERS+="$dns#${RESOLVER} "
done

cat <<EOF
[Resolve]
DNS=$DNS_SERVERS
FallbackDNS=$DNS_SERVERS
Domains=~.
DNSOverTLS=yes
DNSSEC=yes
EOF

```

这个脚本查询 `abc123.cloudflare-gateway.com` 的所有IP地址，并在末尾附加 `#abc123.cloudflare-gateway.com`，以告知 systemd-resolved 使用的 [SNI 名称](https://en.wikipedia.org/wiki/Server_Name_Indication)。由于 DoT 还依赖CA证书，它将从远程证书中获取名称，并与`#`之后提供的名称进行验证。如果不正确，DoT将因不受信任而连接失败。如果未指定SNI名称，DoT将回退到它所拥有的信息——IP地址——以验证证书的有效性。

脚本的输出是 `systemd-resolved` 的配置，其中包含通过 `DNS=` 和 `FallbackDNS=` 设置的 DNS 服务器，使用 `Domains=~.` 将它们作为默认域处理器。使用 `DNSOverTLS=yes`，启用DoT；使用 `DNSSEC=yes` 强制 [DNSSEC](https://en.wikipedia.org/wiki/Domain_Name_System_Security_Extensions)。这样确保全局默认设置安全，且实际使用 `abc123.cloudflare-gateway.com`。

下一步则利用上面创建的脚本配置 `systemd-resolved`：

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d/
generate-dns-config.sh | sudo tee /etc/systemd/resolved.conf.d/cloudflare-private-dot.conf
sudo systemctl restart systemd-resolved

```

路径 `/etc/systemd/resolved.conf.d/` 可用于放置自定义配置，便于发行版更轻松地维护默认配置。使用 `sudo tee …` 可以将配置放入名为 cloudflare-private-dot.conf 的文件中，并在终端中获得相同的输出以进行验证。最后重启 `systemd-resolved`。如果 DNS 配置出现问题，只需删除该文件并重启守护进程即可恢复到默认设置。

## 配置 NetworkManager

要真正利用这个全局配置，必须禁用 NetworkManager 提供的每个接口配置。你可以添加另一个补充配置，但这次是针对  NetworkManager 的：

```bash
sudo mkdir -p /etc/NetworkManager/conf.d/
sudo tee /etc/NetworkManager/conf.d/cloudflare-dot.conf <<EOF
[main]
dns=none
systemd-resolved=false
EOF

```

`dns=none` 告诉 NetworkManager 不要修改 `/etc/resolv.conf`，因为你仍然希望在那里使用 `systemd-resolved` 服务。`systemd-resolved=false` 告诉 NetworkManager 停止与 `systemd-resolved` 通信，因此不再将任何自动获取自接口的 DNS 设置传递给它。

这种配置的缺点是，NetworkManager 中的 DNS 设置现在变得无效。但作为回报，所有接口都将使用全局配置，该配置指向 `abc123.cloudflare-gateway.com`。

想要检查结果是否正确，可以执行命令 `resolvectl`。

```txt
Global
           Protocols: LLMNR=resolve -mDNS +DNSOverTLS DNSSEC=yes/supported
    resolv.conf mode: stub
  Current DNS Server: 162.159.55.66#abc123.cloudflare-gateway.com
         DNS Servers: 162.159.55.77#abc123.cloudflare-gateway.com
                      162.159.36.20#abc123.cloudflare-gateway.com
                      2606:4700:abc::abc1:abc1#abc123.cloudflare-gateway.com
                      2606:4700:abc::abc2:abc2#abc123.cloudflare-gateway.com
Fallback DNS Servers: 162.159.55.66#abc123.cloudflare-gateway.com
                      162.159.55.77#abc123.cloudflare-gateway.com
                      2606:4700:abc::abc1:abc1#abc123.cloudflare-gateway.com
                      2606:4700:abc::abc2:abc2#abc123.cloudflare-gateway.com
          DNS Domain: ~.

Link 2 (enp0s25)
    Current Scopes: DNS LLMNR/IPv4 LLMNR/IPv6
         Protocols: +DefaultRoute +LLMNR -mDNS +DNSOverTLS DNSSEC=yes/supported
Current DNS Server: 1.1.1.1
       DNS Servers: 1.1.1.1 9.9.9.9
        DNS Domain: ~.
     Default Route: yes

```

在这些输出中你会注意到 `enp0s25` 仍然有自己的 DNS 服务器配置。执行 `sudo resolvectl dns enp0s25 ""` 移除它。

之后看起来像这样（不同版本的系统结果有所区别）：

```txt
Global
           Protocols: LLMNR=resolve -mDNS +DNSOverTLS DNSSEC=yes/supported
    resolv.conf mode: stub
  Current DNS Server: 162.159.55.66#abc123.cloudflare-gateway.com
         DNS Servers: 162.159.55.77#abc123.cloudflare-gateway.com
                      162.159.36.20#abc123.cloudflare-gateway.com
                      2606:4700:abc::abc1:abc1#abc123.cloudflare-gateway.com
                      2606:4700:abc::abc2:abc2#abc123.cloudflare-gateway.com
Fallback DNS Servers: 162.159.55.66#abc123.cloudflare-gateway.com
                      162.159.55.77#abc123.cloudflare-gateway.com
                      2606:4700:abc::abc1:abc1#abc123.cloudflare-gateway.com
                      2606:4700:abc::abc2:abc2#abc123.cloudflare-gateway.com
          DNS Domain: ~.

Link 2 (enp0s25)
    Current Scopes: LLMNR/IPv4 LLMNR/IPv6
         Protocols: -DefaultRoute +LLMNR -mDNS +DNSOverTLS
                    DNSSEC=yes/supported
     Default Route: no

```

这之后，如果你想，可以用 Wireshark 抓包验证结果。现在执行 `resolvectl query cloudflare.com` 你应该只会看见加密流量：

```txt
cloudflare.com: 2606:4700::6810:84e5           -- link: enp0s25
                2606:4700::6810:85e5           -- link: enp0s25
                104.16.132.229                 -- link: enp0s25
                104.16.133.229                 -- link: enp0s25

-- Information acquired via protocol DNS in 359.7ms.
-- Data is authenticated: yes; Data was acquired via local or encrypted transport: yes
-- Data from: network

```

## 结论

总的来说，到目前为止，这个方法效果非常好，不再有未加密的 DNS 网络流量。这不仅可以防止基于网络的 DNS 攻击，还能减少信息泄露。这是实现更安全网络通信的重要一步。DNS 几乎是互联网上所有应用的基础，因此即使你甚至没有注意到差异，在最好的情况下，它也有助于为未来更安全的互联网做好准备。请注意，攻击者或网络提供商仍然可以通过收集 SNI 头信息来跟踪您访问的网站，除非客户端和服务器支持ECH。

值得一提的是，这种设置在使用如 Docker 和 Podman 等容器技术时并不能完美运行，因为在使用网络隔离时，容器将无法联系 `systemd-resolved`。不过，这可以通过提供 `--dns` 参数或在配置中设置特定的常规 DNS 服务器来解决。当然，这意味着容器无法从安全 DNS 中受益。

## 检查  DNS 加密状态

- [Browsing Experience Security Check](https://www.cloudflare.com/ssl/encrypted-sni/)

## 相关参考链接

- [Configure DoT on systemd-resolved](https://shivering-isles.com/2021/01/configure-dot-on-systemd-resolved)
- [在 Firefox 上设置 DoH 和 ESNI/ECH，完成加密浏览的最后一块拼图](https://blog.outv.im/2020/firefox-doh-ech-esni/)
- [Cheetsheet：启用ECH](https://blog.outv.im/2025/enabling-ech/)
- [Enabling DNS over TLS on Fedora 42 with systemd-resolved](https://beesley.tech/enabling-dns-over-tls-on-fedora-42-with-systemd-resolved/)

