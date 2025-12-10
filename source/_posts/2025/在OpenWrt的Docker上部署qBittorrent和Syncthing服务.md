---
title: 在OpenWrt的Docker上部署qBittorrent和Syncthing服务
date: 2025-12-10
tags:
  - 年份-2025
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-技术知识
  - 主题-Docker
  - 主题-OpenWrt
  - 主题-Linux
  - 主题-Bittorrent
  - 主题-Syncthing
  - 主题-建站
---

## 部署 qBittorrent 服务

由于 Docker 网口不支持动态 IPv6，我们只能用 `Host` 网络模式部署服务。之后用 Cloudflare Tunnel 反向代理 `http://localhost:8081` 即可。

```bash
#!/bin/sh

docker run \
	--name qbittorrent \
	--network host \
	--restart unless-stopped \
	-e TZ="Asia/Shanghai" \
	-e WEBUI_PORT=8081 \
	-e TORRENTING_PORT=6881 \
	-v /root/qbittorrent:/config \
	-v /root/share/qbittorrent:/downloads \
	-d lscr.io/linuxserver/qbittorrent:latest

```

### qBittorrent 设置

登录使用的账号和密码尽量设置复杂随机些（账号长度不小于 8 位；密码长度不小于 16 位，且含标点符号数字大小写字母），防止别有用心之人搞破坏。以下操作在【Option】窗口中进行。

1. 点击【Behavior】选项卡，设置 Log Files Save path 为 `/config/logs`
2. 点击【Downloads】选项卡，勾选【Merge trackers to existing torrent】，勾选【Copy `.torrent` file to: `/downloads/Seeds`】
3. 点击【Connection】选项卡，在【Peer connection protocol】选择 `TCP`。
4. 点击【Speed】选项卡，在【Global Rate Limits】一栏的【Upload】这里限制上传带宽为 `2048` KiB/s（除非你不怕被运营商认定为 PCDN 而断网）。
5. 点击【BitTorrent】选项卡，在【Encryption mode】选择【Require encryption】强制加密。不管什么流量，加密总归是最好的。然后勾选【Automatically append trackers from URL to new downloads】这一栏，URL 填 `https://cdn.jsdelivr.net/gh/ngosang/trackerslist@master/trackers_all_https.txt`
6. 点击【Web UI】选项卡，在【Ban client after consecutive failures】这里改成 `1`，【ban for】改成 `30`。因为之后设置完 Cloudflare 反向代理，Ban 的 IP 就是回环网卡的 `::1`！所以建议设置一个高强度的账号名和密码。
7. 点击【Advanced】选项卡，设置【Network interface】为 PPPoE 网卡，同时在下一条【Optional IP address to bind to】选择【All IPv6 address】。

## 部署 Syncthing 服务

由于 Docker 网口不支持动态 IPv6，我们只能用 `Host` 网络模式部署服务。之后用 Cloudflare Tunnel 反向代理 `http://localhost:8384` 即可。

如果是**客户端**侧，推荐使用系统的软件包管理器安装并将以下指令加入`~/.bash_profile`文件实现登录自启：`nohup syncthing serve >> /tmp/syncthing.log 2>1&`（或点击App图标）。请勿使用容器部署，可能会出现权限异常。使用 chmod 和 chown 命令可以手动修改文件权限。

```bash
#!/bin/sh

docker run \
    --name syncthing \
    --restart unless-stopped \
    --network host \
    -e TZ="Asia/Shanghai" \
    -v /root/syncthing:/config \
    -v /root/share:/data \
    -d lscr.io/linuxserver/syncthing:latest

```

### Syncthing 设置

按照上面的命令执行进入 Web 管理页面后，点击【Actions —— Logs】里面会警告 UDP Buffer Size 太小会影响 QUIC 带宽。SSH 登录 OpenWrt，执行以下命令永久配置最大缓存限制值:（[来源](https://github.com/quic-go/quic-go/wiki/UDP-Buffer-Sizes)）

```bash
sysctl -w net.core.rmem_max=7500000
sysctl -w net.core.wmem_max=7500000

tee /etc/sysctl.d/syncthing-quic.conf << EOF
net.core.rmem_max=7500000
net.core.wmem_max=7500000
EOF

```

同上一小节，登录使用的账号和密码尽量设置复杂随机些（账号长度不小于 8 位；密码长度不小于 16 位，且含标点符号数字大小写字母），防止别有用心之人搞破坏。以下操作在【Actions —— Settings】窗口中进行。

1. 点击【General】选项卡，设置【Device Name】。
2. 点击【GUI】选项卡，设置【GUI Authentication User】和【GUI Authentication Password】。Syncthing 不像 qBittorrent，不会登录失败封禁 IP，所以需要更高强度的账号密码。
3. 点击【Connections】选项卡，设置【Outgoing Rate Limit (KiB/s)】为 `4096`（除非你不怕被运营商认定为 PCDN 而断网）。
4. 添加自己的设备。

## OpenWrt 防火墙设置

- 【qBittorrent】放行 `wan` 到 `device` 的 IPv6 Only `6881` TCP/UDP 端口。
- 【Syncthing】放行 `wan` 到 `device` 的 IPv6 Only `22000` `21027` 两个 TCP/UDP 端口。

## Cloudflare 反向代理

在 Cloudflare Zero Trust Tunnels 中分别添加 qBittorrent 和 Syncthing 这两个服务的 Tunnel，绑定你的域名。我这里就不多解释了，可以看【[OpenWrt配置Cloudflare隧道](/2025/5/27/OpenWrt%E9%85%8D%E7%BD%AECloudflare%E9%9A%A7%E9%81%93.html)】这篇文章。

