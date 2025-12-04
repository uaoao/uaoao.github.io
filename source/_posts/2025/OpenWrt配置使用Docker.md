---
title: OpenWrt配置使用Docker
date: 2025-04-18
tags:
  - 年份-2025
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-技术知识
  - 主题-Docker
  - 主题-OpenWrt
  - 主题-Linux
---

## 安装

安装全套软件包，建议不要在网页上安装，可能会有一些奇怪的问题。使用以下命令安装全套Docker软件包：

```bash
opkg update
opkg install luci-app-dockerman
reboot

```

## 配置镜像服务器

由于众所周知的原因，某些地区无法直接访问Docker Hub，所以需要配置镜像服务器。而曾经某些高校和企业提供的镜像服务器也由于某些不可抗力因素停止服务。在添加配置之前，先尝试用浏览器打开网站，看是否能正常访问。我这里提供几个截至 ***2025年11月*** 可用的镜像服务器配置和可用的订阅，不保证永久可用。

- 在`/etc/config/dockerd`中配置镜像服务器，**注意缩进整齐**（或者网页 Docker -> Configuration 配置）：

```txt
list registry_mirrors 'https://docker.yomansunter.com'
list registry_mirrors 'https://666860.xyz'
list registry_mirrors 'https://docker-0.unsee.tech'
list registry_mirrors 'https://docker.1ms.run'
list registry_mirrors 'https://docker.1panel.live'
list registry_mirrors 'https://docker.xuanyuan.me'

```

- 订阅网站：

1. <https://github.com/dongyubin/DockerHub>
2. 自己搭建 <https://github.com/cmliu/CF-Workers-docker.io>

测试可用的镜像服务器：以 `https://666860.xyz` 为例子，执行 `curl -v https://666860.xyz` 输出大量内容说明没被墙。也可以 `docker pull 666860.xyz/library/busybox:latest` 拉取一个镜像试试。

## 配置 OpenWrt 防火墙允许 wan 域访问 docker 域

**由于 OpenWrt 的安全策略，如果不配置防火墙，容器内部就无法访问互联网**！！！

1. 登陆 OpenWrt 管理网站，打开 Network —— Firewall —— General Settings 页面，点击编辑【Zones】栏目的 `docker => REJECT` 这一条。

2. 在【Allow forward to destination zones】这里勾选 wan 区域，然后保存并应用，配置效果类似 lan 区域。如图所示：

![OpenWrt配置使用Docker-防火墙](images/2025/OpenWrt配置使用Docker-防火墙.webp)

3. 执行以下命令，测试  `bridge` 接口的容器能否访问互联网：

```bash
docker run --network bridge --rm library/busybox:latest ping -c 4 ip.cn

```

### 自定义接口联网

以上配置可以让 `bridge` 接口联网，接下来我们试试自定义接口。创建一个新桥接接口 `testbr`：

```bash
docker network create --subnet 172.20.20.0/24 --gateway 172.20.20.1 testbr
docker network inspect testbr
# 输出接口相关信息

```

假设接口 ID 前几个字符是 `01b8e`，执行下面的命令显示它在 OpenWrt 中的名称：

```bash
ip a | grep -i 01b8e
# 显示  br-01b8e59a2342 这个接口

```

再创建一个容器，测试使用该接口的容器能否访问互联网：

```bash
docker run --network testbr --ip 172.20.20.22 --rm library/busybox:latest ping -c 4 ip.cn

```

结果是卡了一会儿显示 100% packet loss。因为 Docker 创建接口后不会主动将其加入 `docker` 这个在 OpenWrt 中显示紫色的 ***防火墙域***。所以你会发现使用这个接口的容器全都无法联网，甚至容器之间也不能通信。解决办法很简单（虽然我折腾了很久），就是禁止 Docker 使用 `iptable`（转而使用 `nftable`），然后将刚创建的网卡加入 `docker` 这个紫色的 ***防火墙域***。

1. 进入 OpenWrt 命令行，编辑 `/etc/config/dockerd`，将 `option iptables` 改成 `0`，然后执行 `/etc/init.d/dockerd restart`。
2. 在 OpenWrt 管理页面，进入 Network —— Firewall —— General Settings 页面，找到底部的【Zones】。
3. 编辑紫色的 `docker` 防火墙域，点击【Advanced Settings】选项卡，找到【Covered devices】，点击添加之前创建的接口 `br-01b8e59a2342`。，然后保存并应用。

再创建一个容器，测试使用该接口的容器能否访问互联网：

```bash
docker run --network testbr --ip 172.20.20.22 --rm library/busybox:latest ping -c 4 ip.cn

```

这次就可以正常联网了。测试完毕，删除接口前记得 ***先去之前的防火墙管理页面移除它***。

```
docker network rm testbr

```

记住，以后添加或删除接口（主要是 Docker Compose）都要按照上面的操作修改 `docker` 域的配置。

## 默认桥接接口启用 IPv6（可选）

> 注意：如果你使用 host 模式的反向代理容器（比如Caddy或Nginx），反向代理会将IPv6请求转向IPv4容器内部访问，无需按本节内容配置 IPv6。本文中的IPv6地址是文档专用地址，无法真正联网，仅用于开启IPv6端口。

通常情况下，家庭网络的IPv6前缀在宽带重新拨号后会改变，而且目前 Docker 仍未支持前缀委派自动生成公网IPv6，因此这里的方法仅适用于容器内部不用向互联网发起纯IPv6请求的情况。因为这个方法配置的IPv6地址不是真正的公网IPv6地址，无法访问外部IPv6服务器。当然，容器内部可以正常请求互联网中IPv4地址（配置方法见上一节）。

1. SSH 连接进入 OpenWrt，编辑 `/etc/config/dockerd` 在 `config globals 'globals'` 配置下添加以下选项，**注意缩进整齐**：

```txt
option ipv6 'true'
option fixed_cidr_v6 '2001:db8:1::/64'

```

> 提示：官方配置文件见 [Github](https://github.com/openwrt/packages/blob/master/utils/dockerd/files/etc/config/dockerd)。建议先用官方配置覆盖后再自行配置。

2. 保存并重启路由器，再次 SSH 连接进入 OpenWrt。执行命令 `ip a` 可以看到 `docker0` 这个接口的 `inet6` 新增了上面配置的地址：

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

1. SSH 进入 OpenWrt。编辑 `/etc/config/uhttpd` 文件中 `config uhttpd 'main'` 这个项目中的配置，修改端口到大于1024的端口，比如`2080/2443`，如下所示，**注意缩进整齐**：

```txt
list listen_http '0.0.0.0:2080'
list listen_http '[::]:2080'
list listen_https '0.0.0.0:2443'
list listen_https '[::]:2443'

```

2. 执行 `/etc/init.d/uhttpd restart` 重启 LuCI 管理页面的服务器，以后通过`2080/2443`端口登陆即可。

## 相关参考网址

- [Use IPv6 networking](https://docs.docker.com/engine/daemon/ipv6/)
- [Docker on openwrt](https://embeng.dynv6.net/docker-on-openwrt)
- [Proper network setup with nftables and Docker](https://forum.openwrt.org/t/proper-network-setup-with-nftables-and-docker/187292/4)

