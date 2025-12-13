---
title: RaspberryPi5用作Minecraft服务器
date: 2025-12-13
tags:
  - 年份-2025
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-技术知识
  - 主题-AlmaLinux
  - 主题-Minecraft
  - 主题-建站
  - 主题-RaspberryPi
  - 主题-Beszel
  - 主题-Linux
  - 主题-DDNS
---

自从用上 BPI-R4，将树莓派跑的服务全部搬家重建后，树莓派就被闲置在抽屉里无所事事了。

最近有外出打工的可能性，计划把家里的一切打点好再准备上路修炼。想到以后哪位好友或我自己可能还会想着一起联机玩玩 MC，就把吃灰的 RPI5 掏出来当作服务器使啰。

当时专门买了一个 Argon NEO 5 的外壳，底座最大支持 NVMe 2280 型号的 SSD。正好有一块闲置的 128G 固态，今天掏出来捣鼓了半天，发现开机后老是掉盘，接 USB 启动却没问题。

我一度认为是拓展板硬件坏了，在 TG 群受高人指点才知道，这个硬盘 `3.3V 2.5A` 的功率过高，在 **PCIex1_Gen2** 速率（5GT）上会开机掉盘，而且 Argon 官方给的兼容性列表中都是低功耗硬盘。尝试降级到 **PCIex1_Gen1** 速率（2.5GT）后才正常工作。

## 准备工作

必要：

- OpenWrt 路由器（光猫桥接路由器拨号上网）
- Raspberry Pi 5 (8G) with Official Power Adaptor
- 树莓派 PCIe-NVMe 扩展板
- NVMe 硬盘，不小于 64G，不建议用 TF 卡
- NVMe-USB 硬盘盒，用来刷写系统
- 一台电脑
- Cloudflare 托管有可用域名
- 良好的网络环境和公网 IPv6

可选：

- 显示器
- HDMI 视频线 和 mini HDMI 转接头

## 安装 AlmaLinux 操作系统

AlmaLinux 算是 RHEL 发行版的分支。我很喜欢 RHEL 系列的操作系统，有开箱即用的 Systemd、Firewall 4 和 SELinux 支持，非常稳定、安全、方便且现代。

AlmaLinux 的树莓派版本下载和安装可以参考 [官方文档](https://wiki.almalinux.org/documentation/raspberry-pi.html)。我下载的是无桌面版 `AlmaLinux-10-RaspberryPi-latest.aarch64.raw.xz` 这个文件，下载完成后记得校验哈希值。如果你喜欢用 `dd`，可以解压后直接刷写，否则不要解压。

启动 [Raspberry Pi Imager](https://flathub.org/en/apps/org.raspberrypi.rpi-imager)，选择**树莓派5**，操作系统选择自定义文件，设备选择 USB 外接的 SSD，刷写！

完成后挂载 Boot 分区，修改 `config.txt` 这个文件，其中 `[pi5]` 是新增部分：

```ini
# This file is provided as a placeholder for user options
# AlmaLinux - few default config options
[all]
auto_initramfs=1

[pi5]
usb_max_current_enable=1
dtparam=pciex1_gen=1

[pi4]
arm_boost=1

[all]
# enable serial console
enable_uart=1

```

参数 `usb_max_current_enable=1` 表明启用 USB 最大电流输出；`dtparam=pciex1_gen=1` 用于强制降级 PCIe 速率到 Gen1，防止功耗太高掉盘，如果你用了 PCIe HAT+ 这种有 GPIO 供电的扩展板，可以改成 Gen3 速率（8GT）试试。

接着修改 `/boot/user-data` 文件，这个是系统第一次启动时初始化用户配置的文件：

```yml
#cloud-config
#
# This is default cloud-init config file for AlmaLinux Raspberry Pi image.
#
# If you want additional customization, refer to cloud-init documentation and
# examples. Please note configurations written in this file will be usually
# applied only once at very first boot.
#
# https://cloudinit.readthedocs.io/en/latest/reference/examples.html

hostname: almalinux.local
ssh_pwauth: true

users:
  - name: almalinux
    groups: [ adm, systemd-journal, wheel ]
    sudo: [ "ALL=(ALL) NOPASSWD:ALL" ]
    lock_passwd: false
    passwd: $6$EJCqLU5JAiiP5iSS$wRmPHYdotZEXa8OjfcSsJ/f1pAYTk0/OFHV1CGvcszwmk6YwwlZ/Lwg8nqjRT0SSKJIMh/3VuW5ZBz2DqYZ4c1
    # Uncomment below to add your SSH public keys as YAML array
    #ssh_authorized_keys:
      #- ssh-ed25519 AAAAC3Nz...

```

修改的只有两个地方：一个是 `ssh_pwauth: true`，允许非特权账号密码登录；另一个则是 `groups: [ adm, systemd-journal, wheel ]`，允许 `almalinux` 使用 `sudo` 指令提权。你也可以添加 SSH 公钥登录，我家有专门的 Gateway 所以免了。

插回 SSD，~~上电，开机，轻松秒杀~~！

SSH 登录，用户名 `almalinux`，密码 `almalinux`，登录成功后先修改密码，安装以下软件：

```bash
sudo dnf update

sudo dnf install \
    zstd \
    toolbox \
    bash-color-prompt \
    bash-completion \
    pciutils
```

zstd 提供 `tar` 命令可解压的 `.tar.zst` 格式，如果你不用这个格式压缩文件可以忽略；`toolbox` 提供容器化隔离的终端命令（同时会安装 `podman`），比如说你希望安装 `fastfetch` 但是 AlmaLinux 系统软件源里没有，执行 `toolbox create && toolbox enter` 创建一个容器化的 Fedora 就可以用；`bash-color-prompt` 和 `bash-completion` 方便执行命令；`pciutils` 用于检查 PCIe 信息，比如执行 `sudo lspci -vvv` 检查 SSD 的连接速率是不是 2.5GT x1。

## OpenWrt 绑定 IP 地址

1. 在树莓派中执行 `ip a` 查看 IPv4 和 IPv6 地址。IPv6 只看以 `2` 开头的最短 IP（DHCPv6）。
2. 登录 OpenWrt 管理页面，往下滑到【Active DHCP Leases】一栏， 找到 Hostname 是 `almalinux` 或对应 IPv4 地址的一条记录，点击右侧【Set Static】，先不管IPv6。
3. 进入【Network —— DHCP and DNS】，点击【Static Leases】
4. 点击【Edit】编辑刚才设置为静态的记录：
    - 【Lease time】infinite
    - 【DUID】看末尾后缀对应的IPv6地址
    - 【IPv6-Suffix (hex)】推荐使用 IPv4 后缀，方便记忆和分辨。比如 IPv4 是 `192.168.1.248`，这里就填 `0248`
5. 保存并应用。
6. 重启树莓派，再执行 `ip a`，可以发现 `2` 开头的最短 IP（DHCPv6）末尾现在是 `248`。

## Podman 部署容器

先以非特权用户身份执行命令 `loginctl enable-linger $USER` 以保持登录会话，执行后可以永久启用该功能，即使重新启动也能保持启用状态。使用命令 `ls /var/lib/systemd/linger/` 检查是否生效，文件名就是用户名。

> [参考](https://www.man7.org/linux/man-pages/man1/loginctl.1.html)
>
> - `enable-linger [USER...], disable-linger [USER...]`
>
> 启用/禁用一个或多个用户的用户留存功能。如果为特定用户启用，则会在启动时为该用户创建一个用户管理器，并在用户注销后保留该管理器。这允许未登录的用户运行长时间运行的服务。此设置接受一个或多个用户名或数字 UID 作为参数。如果未指定参数，则启用/禁用调用者会话中用户的留存功能。

上上节内容提到，安装 `toolbox` 会自动安装 `podman`，也可以选择单独安装。接下来就开始部署所需的容器了。保存执行以下几个脚本安装容器（非特权用户）。

### Beszel-agent

```bash
#!/bin/sh

podman run \
  --name beszel-agent \
  --network host \
  -v beszel-agent:/var/lib/beszel-agent \
  -v /run/user/1000/podman/podman.sock:/run/user/1000/podman/podman.sock:ro \
  -v /var/run/dbus/system_bus_socket:/var/run/dbus/system_bus_socket:ro \
  -e TZ="Asia/Shanghai" \
  -e LISTEN=45876 \
  -e HUB_URL="https://monitor.wagada.cc" \
  -e TOKEN="xxxxxxxxx-xxxxxxxxxx-xxxxxx-xxxxx-xxxxx" \
  -e KEY="ssh-ed25519 AAAAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" \
  -d docker.1panel.live/henrygd/beszel-agent:latest

```

具体请参考 [在OpenWrt的Docker上部署Beszel监控](/2025/6/15/在OpenWrt的Docker上部署Beszel监控.html)。我这里部署的是 Agent。Beszel 可以部署大量 Agent，用一个 Hub 集中收集数据。[官方文档](https://beszel.dev/) 提到它支持识别 Podman 套接字和 Systemd 以读取容器和进程信息。不知为何，我这里无法识别（可能与 SELinux 策略有关）。

接着创建 Systemd 服务：

```bash
podman generate systemd --files --name beszel-agent
mv container-beszel-agent.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable container-beszel-agent.service
systemctl --user start container-beszel-agent.service
systemctl --user status container-beszel-agent.service

```

开放本机防火墙端口（如果你在树莓派上部署了 Beszel Hub，请忽略这一步）：

```bash
sudo firewall-cmd --permanent --add-port=45876/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all

```

OpenWrt 也需要设置（如果你在树莓派上部署了 Beszel Hub，请忽略这一步）：

1. 进入 OpenWrt 管理页面，点击【Network —— Firewall】
2. 在【General Settings】选项卡下滑找到【Zones】
3. 编辑紫色的 `docker` 这一条，在【Allow forward to destination zones:】这一栏添加 `lan`
4. 保存并应用

### DDClient

```bash
#!/bin/sh

podman run \
  --name ddclient \
  --network host \
  -e TZ="Asia/Shanghai" \
  -v $PWD/ddclient:/config:z \
  -d lscr.io/linuxserver/ddclient:latest

```

DDClient 是动态 DNS 解析客户端。我的域名托管在 Cloudflare 且家宽只有 IPv6 公网地址，下面的配置按需修改。

编辑 `$PWD/ddclient/ddclient.conf`，找到 `use=if if=xxx # via interfaces` 这一行，修改成以下内容：

```txt
usev6=ifv6,    ifv6=end0       # via interfaces

```

`end0` 是执行 `ip a` 查看到的网卡，默认是 IPv4，这里只用 IPv6。找到 【CloudFlare】这部分的配置，编辑：

```txt
##
## CloudFlare (www.cloudflare.com)
##
protocol=cloudflare,        \
zone=example.com,            \
ttl=1,                      \
login=example@example.com,     \ # Only needed if you are using your global API key. If you are using an API token, set it to "token" (without double quotes).
password=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx,             \ # This is either your global API key, or an API token. If you are using an API token, it must have the permissions "Zone - DNS - Edit" and "Zone - Zone - Read". The Zone resources must be "Include - All zones".
mc.example.com,minecraft.example.com

```

接着创建 Systemd 服务：

```bash
podman generate systemd --files --name ddclient
mv container-ddclient.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable container-ddclient.service
systemctl --user start container-ddclient.service
systemctl --user status container-ddclient.service

```

### Minecraft

```bash
#!/bin/sh

podman run \
  --name minecraft \
  --network podman \
  --dns 223.6.6.6 \
  --dns 223.5.5.5 \
  -p 25565:25565 \
  -v $PWD/minecraft/data:/data:z \
  -v $PWD/minecraft/mods:/mods:z \
  -e TZ="Asia/Shanghai" \
  -e SERVER_NAME="Minecraft Neoforge Server" \
  -e ICON="https://www.minecraft.net/content/dam/minecraftnet/games/minecraft/logos/Homepage_Gameplay-Trailer_MC-OV-logo_300x300.png" \
  -e OVERRIDE_ICON="TRUE" \
  -e FORCE_GAMEMODE="TRUE" \
  -e EULA="TRUE" \
  -e MOTD="Welcome to Minecraft" \
  -e MAX_PLAYERS=50 \
  -e INIT_MEMORY="512M" \
  -e MAX_MEMORY="6G" \
  -e VERSION=1.21.1 \
  -e TYPE="NEOFORGE" \
  -e MODS="/mods" \
  -e DIFFICULTY="hard" \
  -e MODE="survival" \
  -e VIEW_DISTANCE=16 \
  -e MAX_BUILD_HEIGHT=512 \
  -e LEVEL_TYPE="minecraft:large_biomes" \
  -d docker.1panel.live/itzg/minecraft-server:java21-graalvm

```

我的树莓派 5 有 **8G** 内存，如果你的内存只有 4G，建议调低一些参数。其中 `MAX_MEMORY` 不要大于内存的 `90%`。另外，据说有人测试过 `graalvm` 的 JVM 性能优化非常好，~~虽然我看不出来~~。另外还有一些参数可以关闭正版验证，具体参考[容器作者的文档](https://docker-minecraft-server.readthedocs.io/en/latest/)

接着创建 Systemd 服务：

```bash
podman generate systemd --files --name minecraft
mv container-minecraft.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable container-minecraft.service
systemctl --user start container-minecraft.service
systemctl --user status container-minecraft.service

```

开放本机防火墙端口：

```bash
sudo firewall-cmd --permanent --add-port=25565/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all

```

开放 OpenWrt 路由器防火墙端口：

1. 进入 OpenWrt 管理页面，进入【Network —— Firewall】，点击【Traffic Rules】选项卡。
2. 点击添加一条新规则：
    - 【Name】Allow-Minecraft
    - 【Protocol】TCP
    - 【Source zone】wan
    - 【Destination zone】lan
    - 【Destination address】`::248/-64`（这里填树莓派 DHCPv6 IPv6后缀，`-64`用于指示主机后缀）
    - 【Destination port】`25565`
3. 点击【Advanced Settings】，在【Restrict to address family】这一条选【IPv6 Only】。
4. 保存并应用。

## 测试

手机开热点，电脑连热点，检查有没有 IPv6。有就打开 Minecraft，多人联机，添加服务器域名。注意 **服务器端的模组必须与客户端相符**，有些模组不需要客户端/服务端安装，大部分需要客户端和服务端同时安装相同版本。如果模组缺失或版本不符，客户端连接服务器时可能会显示错误【IP 地址簇协议不可用】。建议先不安装模组，原版登录试试。

## Mods

以下是我家服务器和客户端安装的模组列表。

### Server

- 【Carry on】
- 【Create】
- 【Dungeons and taverns】
- 【Dungeons and Taverns Stronghold Overhaul】
- 【Dungeons and Taverns Ancient City Overhaul】
- 【Grave Stone】
- 【Sit】
- 【Touhou Little Maid】
- 【UnionLib】【Antique Atlas】
- 【The Twilight Forest】
- 【Distant Horizons】
- 【Geophilic】
- 【oωo (owo-lib)】【The Aether】
- 【Subsurface】
- 【Traveler's Backpack】
- 【Structory】
- 【Structory: Towers】
- 【Incendium】
- 【Curios API】
- 【Tutta's Doors】
- 【Another Furniture】
- 【Blueprint / Abnormals Core】【Autumnity】
- 【Farmer's Delight】
- 【Immersive Aircraft】
- 【Storage Drawers】
- 【RoadWeaver】

### Client

- 【Apple Skin】
- 【Carry On】
- 【Clean Swing】
- 【Create】
- 【Grave Stone】
- 【Iris】【Sodium】
- 【Just Enough Items】
- 【Sit】
- 【Touhou Little Maid】
- 【UnionLib】【Antique Atlas】
- 【The Twilight Forest】
- 【Distant Horizons】
- 【oωo (owo-lib)】【The Aether】
- 【Traveler's Backpack】
- 【Curios API】
- 【Tutta's Doors】
- 【Another Furniture】
- 【Blueprint / Abnormals Core】【Autumnity】
- 【Farmer's Delight】
- 【Immersive Aircraft】
- 【Storage Drawers】
- 【RoadWeaver】
- 【I18nUpdateMod】

## 相关参考链接

- [Fixing 'Permission denied' Errors When Mounting Volumes in Rootless Podman](https://libraibex.com/posts/podman-volumes-permission-denied/)

