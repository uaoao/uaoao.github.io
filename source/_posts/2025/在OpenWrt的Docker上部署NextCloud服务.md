---
title: 在OpenWrt的Docker上部署NextCloud服务
date: 2025-6-8
tags:
  - 2025年
  - OpenWrt
  - Docker
  - NextCloud
  - 建站
  - 配置
  - 教程
  - Linux
---


## 前言

自从买了BPI-R4以来，我一直想要在这路由器上面实现网盘功能。以前的实现方式是用RaspberryPi5加上内网穿透SSHFS和Linux多用户配置实现网盘功能和数据共享。这种实现方式不够安全，但简单且不需要太多学习。

最近把RaspberryPi5拿来测试其他功能了，所以不能再像之前部署简单的网盘服务，更何况老早以前花一千多买的BPI-R4只实现个宽带拨号、路由和DDNS未免也太大材小用。于是最近想尽办法在上面部署一个能够在浏览器访问的网盘服务。

我尝试了很多方案，比如[AList](https://alistgo.com/)、[SFTPGo](https://sftpgo.com/)、[FileBroswer](https://filebrowser.org/)等等，都不太满意。我认为最主要能实现以下几个功能：

1. 网页端文件在线浏览。比如点击播放视频、音乐等。
2. 多用户。对每个用户都能有自己的空间和共享空间。
3. 文件共享。能共享文件，通过内部共享或生成网页链接。
4. WebDAV。一些App需要WebDAV备份，很实用的功能。
5. 网页端的管理面板。能全局设置用户和资源配置。
6. 对用户而言简单易用、看起来舒适。
7. 多协议支持。支持SFTP、FTPS、SMB、NFS等协议，能够无需安装额外软件直接通过操作系统挂载网盘。

SFTPGo支持的功能除了1和6。AList支持的功能除了1、3、6、7。FileBroswer跟AList差不多，还比其支持的内部协议少。而[NextCloud](https://nextcloud.com/)支持的功能只排除7。多协议支持并非必须，只要多平台都有支持的客户端或原生支持某个协议就行。在这些协议中，WebDAV是主流平台Windows10/Linux/MacOS的资源管理器都原生支持的协议，而Android有NextCloud官方App以及其他仅支持WebDAV备份的App。因此我最终决定使用NextCloud部署网盘服务器。


## 折腾失败的经历

最初我使用的是[NextCloud AIO](https://github.com/nextcloud/all-in-one)来部署在OpenWrt的Docker上，研究了好几天。

先是为了实现IPv6公网能够访问容器而修改Docker默认的`bridge`配置以支持IPv6协议，后来发现用[Caddy](https://hub.docker.com/_/caddy)反向代理，选择host网络模式，外网IPv4和IPv6均可通过Caddy访问到默认`bridge`网卡上的容器，不需要再折腾Docker IPv6。

然后发现修改默认的`bridge`网卡无法应用到自动创建的网卡上，而且除默认`bridge`网卡外创建的网卡都无法在容器内部访问互联网。有关容器内部访问互联网的问题，在[OpenWrt配置使用Docker](https://uaoao.github.io/2025/4/18/OpenWrt配置使用Docker.html)这篇文章中有说明。

再然后研究ARM架构部署NextCloud AIO时NextCloud-Redis容器遇到的问题。先是需要用`sysctl`修改个什么参数，接着想办法强制在Redis容器中添加配置文件处理 `ARM64-COW-BUG`。这个还不是最麻烦的，可以想办法解决。最麻烦的是前文那个自定义网卡无法联网的问题，完全摸不着头脑。

最后我放弃使用 NextCloud AIO 了，折腾地近乎绝望。最后只能自己手动一个个地安装服务组件。接下来写的就是如何部署NextCloud服务。

## 部署 NextCloud 服务

本文不使用 Docker Compose。因为在OpenWrt系统中，除了Bridge和host网卡外其他使用自定义Docker网卡的容器都无法正常联网。

在部署服务之前，你需要先配置好以下内容：

- 一个可用的长期域名，比如 `example.com`
- DDNS 绑定 IPv6 域名，比如 `nextcloud.example.com`（参考 [OpenWrt配置Cloudflare-DDNS服务](https://uaoao.github.io/2025/5/31/OpenWrt%E9%85%8D%E7%BD%AECloudflare-DDNS%E6%9C%8D%E5%8A%A1.html)
- Cloudflare Tunnel 内网穿透绑定域名，比如 `nextcloudcf.example.com`（参考 [OpenWrt配置Cloudflare隧道](https://uaoao.github.io/2025/5/27/OpenWrt%E9%85%8D%E7%BD%AECloudflare%E9%9A%A7%E9%81%93.html)）
- Docker 配置完（参考 [OpenWrt配置使用Docker](https://uaoao.github.io/2025/4/18/OpenWrt配置使用Docker.html)）

### 部署 MySQL 容器

- 账号：`root`
- 密码：`change_me` **请修改成自己的密码**
- 数据库：`nextcloud`
- 端口：`3306`
- IPv4 地址：见 OpenWrt 管理页面 Docker —— Container —— Network列表

复制以下内容保存并执行，部署 MySQL。

```bash
#!/bin/sh

docker run \
    --name nextcloud-mysql \
    --network bridge \
    --restart unless-stopped \
    -v /root/nextcloud/mysql:/var/lib/mysql \
    -e TZ=Asia/Shanghai \
    -e MYSQL_DATABASE=nextcloud \
    -e MYSQL_ROOT_PASSWORD=change_me \
    -d library/mysql:latest
```

### 部署 NextCloud 容器

这里使用的是 [linuxserver/nextcloud](https://hub.docker.com/r/linuxserver/nextcloud)。这个容器比 [library/nextcloud](https://hub.docker.com/_/nextcloud/) 安全性更强，内置 NGINX，默认提供HTTPS。

复制以下内容保存并执行，部署 Nextcloud。

```bash
#!/bin/sh

docker run \
	--name nextcloud \
	--network bridge \
	--restart unless-stopped \
	-p 2020:443 \
	--dns 223.6.6.6 \
	--dns 223.5.5.5 \
	-e TZ=Asia/Shanghai \
	-v /root/nextcloud/data:/data \
	-v /root/nextcloud/config:/config \
	-d linuxserver/nextcloud:latest
```

### 配置 OpenWrt 防火墙

> 注意：如果你的运营商封锁了IPv6地址的80/443端口，那么只能使用自定义端口，或者只用Cloudflare Tunnel内网穿透。保险起见，最好是家宽公网IPv6 + 内网穿透。

1. 登陆 OpenWrt 管理网页，如果先前修改过端口，则用修改后的端口登陆。
2. 进入 Network —— Firewall —— Traffic Rules 选项卡，添加一条规则：

- General Settings：
  - Name：Allow-HTTP/S
  - Protocol：TCP、UDP
  - Source zone：wan
  - Destination zone：Device（input）
  - Destination port：80 443
  - Action：accept
- Advanced Settings：
  - Restrict to address family：IPv6 only

3. 保存并应用。

### 部署 Caddy 容器

先创建 `/root/caddy/Caddyfile` 文件，将以下内容保存，域名和端口替换成自己的。我不熟悉配置反向代理，所以这里就简单地配置必要部分。

```txt
nextcloud.example.com:80 {
	redir * https://nextcloud.example.com:443 301
}

nextcloudcf.example.com:80 {
	redir * https://nextcloudcf.example.com:443 301
}

nextcloud.example.com:443 nextcloudcf.example.com:443 {
	reverse_proxy https://localhost:2020 {
		transport http {
			tls
			tls_insecure_skip_verify
		}
	}
}

example.com {
	respond 404
}
```

然后复制以下内容保存并执行，部署 Caddy。

```bash
#!/bin/sh

docker run \
	--name caddy \
	--restart unless-stopped \
	--network host \
	-v /root/caddy/Caddyfile:/etc/caddy/Caddyfile \
	-v /root/caddy/config:/config \
	-v /root/caddy/data:/data \
	-v /root/caddy/srv:/srv \
	-e TZ=Asia/Shanghai \
	-d library/caddy:latest
```

## 配置 NextCloud 服务

1. 用浏览器打开网站 `https://nextcloud.example.com` 或者 `https://nextcloudcf.example.com`。如果加载失败，请检查 Caddy 日志。

2. 填写注册管理员账号和密码，在下方选择 MySQL 数据库，填写账号、密码、MySQL容器内部的IP地址、数据库名称。然后点击安装。

3. 安装应用页面可以跳过，也可以安装好使用。

我第一次部署的时候可以正常安装应用，后面不知怎么总是显示网络异常。光是排查这个问题就非费了大量时间，添加了自定义DNS，OpenWrt系统因此重置过，登陆进容器内部用 `curl https://nextcloud.com` 能正常获取数据，但是网页端依旧网络异常。网上找到一篇文章似乎说明可能是 nextcloud 的问题：[Log flooded with "dns_get_record(): A temporary server error occurred"](https://github.com/nextcloud/server/issues/34205)。安装应用估计没法使用了，挺失望的。

4. 当首次用某个域名登陆并配置好数据库后，该域名将被设置为唯一可以访问的域名，但是我们有两个域名都需要访问。编辑 `/root/nextcloud/config/www/nextcloud/config/config.php`，如下所示找到 `trusted_domains` 这个参数，添加第二个域名。

```txt
  'trusted_domains' => 
  array (
    0 => 'nextcloud.example.com',
    1 => 'nextcloudcf.example.com',
  ),
```

至此，基本配置就结束了。之后通过网页端管理员进行配置即可，更高级的配置我还不会，等学会了再说。

## 检查网站安全性

打开 [Nextcloud Security Scan](https://scan.nextcloud.com/) 填入域名测试。我这里IPv6域名测试结果是 **A+**，而Cloudflare域名测试结构是 **A**。

## 相关参考链接

- [Nextcloud Server Doc](https://docs.nextcloud.com/)
- [Caddy Docker以及配置示例](https://blog.03k.org/post/caddy-docker.html)
