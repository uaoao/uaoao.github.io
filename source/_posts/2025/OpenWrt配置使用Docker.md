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
opkg install luci-app-dockerman kmod-macvlan
reboot
```

## 配置镜像服务器

由于众所周知的原因，某些地区无法直接访问Docker Hub，所以需要配置镜像服务器。而曾经某些高校和企业提供的镜像服务器也由于某些不可抗力因素停止服务。我这里提供几个截至2025年6月可用的镜像服务器配置和可用的订阅，不保证永久可用。

在添加配置之前，先尝试用浏览器打开网站，看是否能正常访问。

- 在`/etc/config/dockerd`中配置镜像服务器，**注意缩进整齐**（或者网页 Docker -> Configuration 配置）：

```txt
list registry_mirrors 'https://lispy.org'
list registry_mirrors 'https://docker.xiaogenban1993.com'
list registry_mirrors 'https://docker.yomansunter.com'
list registry_mirrors 'https://666860.xyz'
list registry_mirrors 'https://a.ussh.net'
list registry_mirrors 'https://docker.1ms.run'
list registry_mirrors 'https://dytt.online'
list registry_mirrors 'https://docker.1panel.live'
list registry_mirrors 'https://dockerproxy.com'
```

- 订阅网站：

1. https://github.com/dongyubin/DockerHub
2. https://github.com/cmliu/CF-Workers-docker.io

## 配置 OpenWrt 防火墙允许 wan 区域转发进入 docker0

**由于OpenWrt的安全策略，如果不配置防火墙，用默认 bridge 网卡的容器内部就无法访问互联网！！！**

1. 登陆 OpenWrt 管理网站，打开 Network —— Firewall —— General Settings 页面，点击编辑 Zones 栏目的 `docker => REJECT` 这一条。

2. 在 Allow forward to destination zones 这里勾选 wan 区域，然后保存并应用，配置效果类似 lan 区域。如图所示：

![OpenWrt配置使用Docker-防火墙](images/OpenWrt配置使用Docker-防火墙.webp)

3. 执行以下命令，测试容器是否能访问互联网：

```bash
docker run --rm library/busybox:latest ping -c 4 www.bilibili.com
```

> 注意：这个操作只能让默认 bridge 网卡上的容器访问互联网。如果你使用 Docker Compose 创建容器，或者创建一个自定义的Docker网卡，这些情况下容器内部都无法访问互联网，需要手动修改`iptable/nftable`来转发流量。这样做很麻烦而且非常不安全。如果你不知道自己执行的命令修改了什么东西，就**不要执行**。

## 默认桥接网卡启用 IPv6

> 注意：如果你使用 host 模式的反向代理容器（比如Caddy或Nginx），反向代理会将IPv6请求转向IPv4容器内部访问，无需按本节内容配置 IPv6。本文中的IPv6地址是文档专用地址，无法真正联网，仅用于开启IPv6端口。

通常情况下，家庭网络的IPv6前缀在宽带重新拨号后会改变，而且目前 Docker 仍未支持前缀委派自动生成公网IPv6，因此这里的方法仅适用于容器内部不用向互联网发起纯IPv6请求的情况。因为这个方法配置的IPv6地址不是真正的公网IPv6地址，无法访问外部IPv6服务器。当然，容器内部可以正常请求互联网中IPv4地址（配置方法见上一节）。

1. SSH 连接进入 OpenWrt，编辑 `/etc/config/dockerd` 在 `config globals 'globals'` 配置下添加以下选项，**注意缩进整齐**：

```txt
option ipv6 'true'
option fixed_cidr_v6 '2001:db8:1::/64'
```

> 提示：官方配置文件见 [Github](https://github.com/openwrt/packages/blob/master/utils/dockerd/files/etc/config/dockerd)。建议先用官方配置覆盖后再自行配置。

2. 保存并重启路由器，再次 SSH 连接进入 OpenWrt。执行命令 `ip addr` 可以看到 `docker0` 这个接口的 `inet6` 新增了上面配置的地址：

```txt
11: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP 
    link/ether 02:44:a2:b2:1d:53 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 2001:db8:1::1/64 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::44:a2ff:feb2:1d53/64 scope link 
       valid_lft forever preferred_lft forever
```

### 测试 lan 区域内对 docker0 IPv6 的可访问性

1. 执行以下命令，创建简易 HTTP 服务器：

```bash
docker run --rm -d -p 80:80 traefik/whoami
```

2. 在 OpenWrt 本机执行 `curl http://[::1]:80` 和 `curl [2001:db8:1::1]:80`，执行成功，结果如下：

```txt
Hostname: 9d5447adacfd
IP: 127.0.0.1
IP: ::1
IP: 172.17.0.3
IP: 2001:db8:1::242:ac11:3
IP: fe80::42:acff:fe11:3
RemoteAddr: [2001:db8:1::1]:37394
GET / HTTP/1.1
Host: [::1]             # 或者显示 [2001:db8:1::1]
User-Agent: curl/8.10.1
Accept: */*
```

可以看到，OpenWrt 本机可以正常访问容器中的 IPv6 地址。

3. 获取 OpenWrt lan 接口的IPv6地址（比如 `2407:342b:2af1:cb42::1`），然后在局域网 lan 中用一台电脑执行 `curl http://[2407:342b:2af1:cb42::1]:80`，执行成功，结果如下：

```txt
Hostname: 9d5447adacfd
IP: 127.0.0.1
IP: ::1
IP: 172.17.0.3
IP: 2001:db8:1::242:ac11:3
IP: fe80::42:acff:fe11:3
RemoteAddr: [2001:db8:1::1]:44902
GET / HTTP/1.1
Host: [2407:342b:2af1:cb42::1]
User-Agent: curl/8.11.1
Accept: */*
```

可以看到，OpenWrt 能自动识别静态地址 `2001:db8:1::1` 并且局域网 lan 中的设备能成功通过 IPv6 访问到 Docker 容器中的网站。

### 测试 wan 区域内对 docker0 IPv6 的可访问性

1. 与上小节相同，执行以下命令，创建简易 HTTP 服务器，已存在则无需创建：

```bash
docker run --rm -d -p 80:80 traefik/whoami
```

2. 登陆 OpenWrt 管理网页，进入 Network —— Firewall —— Traffic Rules 页面，添加一条新规则：

- General Settings：
  - 名称：Allow-HTTP-80
  - 协议：TCP
  - 源区域：wan
  - 目的区域：Device(input)
  - 目的端口：80

- Advanced Settings：
  - 限制地址簇：IPv6 only

其他内容填写默认值，保存并应用。

3. 按照 [OpenWrt配置Cloudflare-DDNS服务](https://uaoao.github.io/2025/5/31/OpenWrt%E9%85%8D%E7%BD%AECloudflare-DDNS%E6%9C%8D%E5%8A%A1.html) 文章配置 DDNS 服务绑定一个域名。如果暂时不想配置域名，可以获取 wan_6 或 lan 接口的 IPv6 地址。

4. 用手机开热点让电脑连上执行 `curl http://[IPv6 公网地址或域名]:80`，或者直接用手机浏览器访问 `http://[IPv6 公网地址或域名]:80`。执行成功，浏览器访问结果如下：

```txt
Hostname: 9d5447adacfd
IP: 127.0.0.1
IP: ::1
IP: 172.17.0.3
IP: 2001:db8:1::242:ac11:3
IP: fe80::42:acff:fe11:3
RemoteAddr: [2001:db8:1::1]:54588
GET / HTTP/1.1
Host: [IPv6 公网地址或域名]
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:139.0) Gecko/20100101 Firefox/139.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.5
Connection: keep-alive
Priority: u=0, i
Upgrade-Insecure-Requests: 1
```

可以看到，OpenWrt 能自动识别静态地址 `2001:db8:1::1` 并且互联网 wan 中的设备能也能通过 IPv6 访问到 Docker 容器中的网站。

## 修改 LuCI 管理页面的端口

默认安装 OpenWrt 时会启用`80`和`443`端口用于登陆 LuCI 网页管理界面。当我们需要反向代理来使用这两个端口时，就需要将 LuCI 的端口改到其他位置。这里提供修改的方法。

1. SSH 进入 OpenWrt。编辑 `/etc/config/uhttpd` 文件中 `config uhttpd 'main'` 这个项目中的配置，修改端口到大于1024的端口，比如`2000/2001`，如下所示，**注意缩进整齐**：

```txt
list listen_http '0.0.0.0:2000'
list listen_http '[::]:2000'
list listen_https '0.0.0.0:2001'
list listen_https '[::]:2001'
```

2. 执行 `/etc/init.d/uhttpd restart` 重启 LuCI 管理页面的服务器，以后通过`2000/2001`端口登陆即可。

## 相关参考网址

- [Use IPv6 networking](https://docs.docker.com/engine/daemon/ipv6/)
- [Docker on openwrt](https://embeng.dynv6.net/docker-on-openwrt)
