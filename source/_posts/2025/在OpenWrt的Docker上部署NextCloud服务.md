---
title: 在OpenWrt的Docker上部署NextCloud服务
date: 2025-6-8
tags:
  - 年份-2025
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-技术知识
  - 主题-OpenWrt
  - 主题-Linux
  - 主题-Docker
  - 主题-建站
  - 主题-NextCloud
---

自从买了 BPI-R4 以来，我一直想要在这路由器上面实现网盘功能。以前的实现方式是用 Raspberry Pi 5 加上内网穿透 SSHFS 和 Linux 多用户配置实现网盘功能和数据共享。这种实现方式不够安全，但简单且不需要太多学习。

最近把 Raspberry Pi 5 拿来测试其他功能了，所以不能再像之前部署简单的网盘服务，更何况老早以前花一千多买的 BPI-R4 只实现个宽带拨号、路由和 DDNS 未免也太大材小用。于是最近想尽办法在上面部署一个能够在浏览器访问的网盘服务。

我尝试了很多方案，比如[AList](https://alistgo.com/)、[SFTPGo](https://sftpgo.com/)、[FileBroswer](https://filebrowser.org/)等等，都不太满意。我认为最主要能实现以下几个功能：

1. 网页端文件在线浏览。比如点击播放视频、音乐等。
2. 多用户。对每个用户都能有自己的空间和共享空间。
3. 文件共享。能共享文件，通过内部共享或生成网页链接。
4. WebDAV。一些 App 需要 WebDAV 备份，很实用的功能。
5. 网页端的管理面板。能全局设置用户和资源配置。
6. 对用户而言简单易用、看起来舒适。
7. 多协议支持。支持SFTP、FTPS、SMB、NFS等协议，能够无需安装额外软件直接通过操作系统挂载网盘。

SFTPGo 支持的功能除了 1 和 6。AList支持的功能除了 1、3、6、7。FileBroswer 跟 AList 差不多，还比其支持的内部协议少。而 [NextCloud](https://nextcloud.com/) 支持的功能只排除 7。多协议支持并非必须，只要多平台都有支持的客户端或原生支持某个协议就行。在这些协议中，WebDAV 是主流平台 Windows10/Linux/MacOS 的资源管理器都原生支持的协议，而 Android 有 NextCloud 官方 App 以及其他仅支持 WebDAV 备份的 App。因此我最终决定使用NextCloud部署网盘服务器。


## 折腾失败的经历

最初我使用的是[NextCloud AIO](https://github.com/nextcloud/all-in-one)来部署在OpenWrt的Docker上，研究了好几天。

先是为了实现IPv6公网能够访问容器而修改 Docker 默认的 `bridge` 配置以支持 IPv6 协议，后来发现用[Caddy](https://hub.docker.com/_/caddy)反向代理，选择 host 网络模式，外网 IPv4 和 IPv6 均可通过Caddy访问到默认 `bridge` 接口上的容器，不需要再折腾 Docker IPv6。

然后发现修改默认的 `bridge` 接口无法应用到自动创建的接口上，而且除默认 `bridge` 接口外的自定义接口都无法在容器内部访问互联网。

再然后研究 ARM 架构部署 NextCloud AIO 时 NextCloud-Redis 容器遇到的问题。先是需要用 `sysctl` 修改个什么参数，接着想办法强制在 Redis 容器中添加配置文件处理 `ARM64-COW-BUG`。这个还不是最麻烦的，可以想办法解决。最麻烦的是前文那个自定义接口无法联网的问题，完全摸不着头脑。

最后我放弃使用 NextCloud AIO 了，折腾地近乎绝望。最后只能自己手动一个个地安装服务组件。接下来写的就是如何部署 NextCloud 服务。

## 部署 NextCloud 服务

~~本文不使用 Docker Compose。因为在 OpenWrt 系统中，除了 Bridge 和 host 外其他使用自定义接口的容器都无法正常联网。~~ 已找到解决办法，有关容器内部访问互联网的问题，在[OpenWrt配置使用Docker](/2025/4/18/OpenWrt配置使用Docker.html)这篇文章中有说明。本文于 **2025年12月** 更新后使用自定义接口 `br-2020` 连接服务。

在部署服务之前，你需要先配置好以下内容：

- 一个可用的长期域名，比如 `example.com`
- DDNS 绑定 IPv6 域名，比如 `nextcloud.example.com`（参考 [OpenWrt配置Cloudflare-DDNS服务](/2025/5/31/OpenWrt%E9%85%8D%E7%BD%AECloudflare-DDNS%E6%9C%8D%E5%8A%A1.html)
- Cloudflare Tunnel 内网穿透绑定域名，比如 `nctun.example.com`（参考 [OpenWrt配置Cloudflare隧道](/2025/5/27/OpenWrt%E9%85%8D%E7%BD%AECloudflare%E9%9A%A7%E9%81%93.html)）
- Cloudflare Proxy 模式绑定域名，比如 `nextcloud.cf.example.com`，支持用户访问IPv4转IPv6。注意，Cloudflare的SSL/TLS加密模式应当设置为**Full**，否则浏览器会显示重定向次数过多的错误。
- Docker 配置完（参考 [OpenWrt配置使用Docker](/2025/4/18/OpenWrt配置使用Docker.html)）

> 注意：Cloudflare 免费用户代理的网站在文件传输上有 100MB 限制，超过限制会临时中断连接。幸好这方面 NextCloud 网页端能自动断点续传，客户端没测试过不清楚。

- 使用以下命令创建自定义接口 `br-2020`:

```bash
docker network create --subnet 172.20.20.0/24 --gateway 172.20.20.1 br-2020

```

### 部署 MySQL 容器

> 注意：如果你只是个人使用，SQLite 完全够用，不必部署独立的数据库服务浪费系统资源。

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
    --network br-2020 \
    --ip 172.20.20.56 \
    --restart unless-stopped \
    -e TZ=Asia/Shanghai \
    -e MYSQL_DATABASE=nextcloud \
    -e MYSQL_ROOT_PASSWORD=change_me \
    -v /root/nextcloud/mysql:/var/lib/mysql \
    -d library/mysql:latest

```

### 部署 NextCloud 容器

这里使用的是 [linuxserver/nextcloud](https://hub.docker.com/r/linuxserver/nextcloud)。这个容器比 [library/nextcloud](https://hub.docker.com/_/nextcloud/) 安全性更强，内置 NGINX，许多配置已经写好了，还默认提供自签证书。

复制以下内容保存并执行，部署 Nextcloud。其中 `-v /mnt/sda1:/mnt/sda1:ro` 的作用是便于读取操作系统挂载的外部设备，为了防止误操作删除数据，这里就设置成只读；`-v /root/share:/mnt/share` 方便与宿主机交换文件（注意权限问题）。

```bash
#!/bin/sh

docker run \
	--name nextcloud \
	--network br-2020 \
	--ip 172.20.20.2 \
	--restart unless-stopped \
	--dns 223.6.6.6 \
	--dns 223.5.5.5 \
	-e TZ=Asia/Shanghai \
	-v /root/nextcloud/data:/data \
	-v /root/nextcloud/config:/config \
	-v /mnt/sda1:/mnt/sda1:ro \
	-v /root/share:/mnt/share \
	-d linuxserver/nextcloud:latest

```

### 配置 OpenWrt 防火墙

> 注意：如果你的运营商封锁了IPv6地址的80/443端口，那么只能使用自定义端口，或者只用Cloudflare Tunnel内网穿透。保险起见，最好是家宽公网IPv6 + 内网穿透。

1. 登陆 OpenWrt 管理网页，如果先前修改过端口，则用修改后的端口登陆。
2. 进入 Network —— Firewall —— Traffic Rules 选项卡，添加一条规则：

- 【General Settings】
  - 【Name】Allow-HTTP/S
  - 【Protocol】TCP、UDP
  - 【Source zone】wan
  - 【Destination zone】Device（input）
  - 【Destination port】80 443
  - 【Action】accept
- 【Advanced Settings】
  - 【Restrict to address family】IPv6 only

3. 保存并应用。

### 只读挂载外部存储器

1. 在 OpenWrt 中安装软件包 `block-mount`，然后重新登录网页端。
2. 插入存储器。
3. 打开 System —— Mount Points，找到【Mount Points】这一小节。
4. 如果没有显示你刚插入的设备，点击【Add】，在【UUID】这一行找到你的存储器。
5. 挂载点填先前部署 Nextcloud 容器时设置的位置 `/mnt/sda1`。
6. 在【Advanced Settings】选项卡【Mount Options】一行填入 `ro`，表示只读挂载。（如果你希望容器只读而宿主机可写则不填）
7. 保存并应用。

### 部署 Caddy 容器

先创建 `/root/caddy/etc/Caddyfile` 文件，将以下内容保存，域名和端口替换成自己的。我不熟悉配置反向代理，所以这里就照葫芦画瓢简单地配置必要部分。`nctun.example.com` 这个域名本身就是被代理的状态，因此不必写在配置里。二级域名返回 `404` 是为了留作以后配置其他服务做主页。

```txt
nextcloud.cf.example.com {
	reverse_proxy https://172.20.20.2:443 {
		transport http {
			tls
			tls_insecure_skip_verify
		}
	}
}

nextcloud.example.com {
	reverse_proxy https://172.20.20.2:443 {
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
	--cap-add NET_ADMIN \
	-v /root/caddy/etc:/etc/caddy \
	-v /root/caddy/config:/config \
	-v /root/caddy/data:/data \
	-v /root/caddy/srv:/srv \
	-e TZ=Asia/Shanghai \
	-d library/caddy:latest

```

### 配置 Cloudflare 隧道

具体可以看 [OpenWrt配置Cloudflare隧道](/2025/5/27/OpenWrt配置Cloudflare隧道.html) 这篇文章，这里注意修改几个设置：

【服务类型】HTTPS
【URL】`172.20.20.2:443`
【TLS 设置 —— 无 TLS 验证】启用
【TLS 设置 —— HTTP2 连接】启用

## 配置 NextCloud 服务

1. 用浏览器打开网站 `https://nextcloud.example.com` 或者 `https://nextcloud.cf.example.com` 或者 `https://nctun.example.com`。如果加载失败，请检查 Caddy 日志（`nextcloud.example.com`是IPv6域名，客户端必须有IPv6才能访问）。
2. 填写注册管理员账号和密码，在下方 ~~选择 MySQL 数据库，填写账号、密码、MySQL容器内部的IP地址、数据库名称~~ 选择默认的 SQLite（个人使用足够了）。然后点击安装。
3. 安装应用页面可以跳过，也可以安装好使用。

我第一次部署的时候可以正常安装应用，后面不知怎么总是显示网络异常。光是排查这个问题就非费了大量时间，添加了自定义DNS，OpenWrt系统因此重置过，登陆进容器内部用 `curl https://nextcloud.com` 能正常获取数据，但是网页端依旧网络异常。后来查了大量资料才发现是进入应用商店会请求一个 20MB 的文本文件，所以超时了。建议直接在配置文件中添加 `'appstoreenabled' => false,` 以禁用商店，手动添加应用。

4. 当首次用某个域名登陆并配置好数据库后，该域名将被设置为唯一可以访问的域名，但是我们有两个域名都需要访问。编辑 `/root/nextcloud/config/www/nextcloud/config/config.php`，如下所示找到 `trusted_domains` 这个参数，添加第二个域名。其他配置参考了官方反向代理的文档。

```txt
  'appstoreenabled' => false,
  'trusted_domains' =>
  array (
    0 => 'nextcloud.example.com',
    1 => 'nctun.example.com',
    2 => 'nextcloud.cf.example.com',
  ),
  'trusted_proxies' =>
  array (
    0 => '172.20.20.1',
  ),
  'forwarded_for_headers' => 'HTTP_X_FORWARDED_FOR',
  'overwriteprotocol' => 'https',
  'enable_previews' => true,
```

### 安装/启用插件

未启用的插件，建议启用：

- 【External storage support】记得在 Nextcloud 设置里挂载外接存储器和宿主机共享的文件夹
- 【Suspicious Login】
- 【Two-Factor Authentication via Nextcloud notification】
- 【Auditing / Logging】

别人制作的插件，现在我都不用了（因为用不上，而且大版本一更新全炸）：

- Music
- Team folders
- Bookmarks
- Draw.io
- Link editor
- Checksum
- Metadata
- External sites
- Share Review
- Talk
- News
- iFrame Widget
- Contacts
- Mail
- EPUB Viewer
- Announcement center
- Tables
- Cookbook
- Notes
- Two-Factor Email
- Two-Factor Admin Support
- Memories
- User migration

至此，基本配置就结束了。之后通过网页端管理员进行配置即可，更高级的配置我还不会，等学会了再说。

## 检查网站安全性

打开 [Nextcloud Security Scan](https://scan.nextcloud.com/) 填入域名测试。我这里 IPv6 域名测试结果是 **A+**，而 Cloudflare 域名测试结构是 **A**。

## 相关参考链接

- [Nextcloud Server Doc](https://docs.nextcloud.com/)
- [Caddy Docker以及配置示例](https://blog.03k.org/post/caddy-docker.html)
- [State of SQLite support](https://help.nextcloud.com/t/state-of-sqlite-support/85283)

