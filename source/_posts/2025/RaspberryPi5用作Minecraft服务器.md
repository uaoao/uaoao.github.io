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
- NVMe 硬盘、TF 卡各一个。TF 卡（不小于16G）用于备份存档，不建议装系统
- NVMe-USB 硬盘盒，用来刷写系统
- 一台计算机
- Cloudflare 托管有可用域名
- 良好的网络环境和公网 IPv6

可选：

- 显示器
- HDMI 视频线 和 Micro HDMI 转接头

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

参数 `usb_max_current_enable=1` 表明启用 USB 最大电流输出；`dtparam=pciex1_gen=1` 用于强制降级 PCIe 速率到 Gen1，防止功耗太高掉盘，如果你用了 PCIe HAT+ 这种有 GPIO 供电的扩展板，可以改成 Gen3 速率（8GT）试试（前提是 SSD 支持 Gen3，现在大部分是Gen3 和 Gen4）。

接着修改 `user-data` 文件，这个是系统第一次启动时初始化用户配置的文件：

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

登录 OpenWrt 管理页面查看树莓派的 IP 地址，然后 SSH 登录。用户名 `almalinux`，密码 `almalinux`，登录成功后先执行 `passwd` 修改密码，安装以下软件：

```bash
sudo dnf update

sudo dnf install \
    zstd \
    toolbox \
    bash-color-prompt \
    bash-completion \
    pciutils

```

zstd 提供 `tar` 命令可解压的 `.tar.zst` 格式，如果你不用这个格式压缩文件可以忽略；`toolbox` 提供容器化隔离的终端命令（同时会安装 Podman），比如说你希望安装 `fastfetch` 但是 AlmaLinux 系统软件源里没有，执行 `toolbox create && toolbox enter` 创建一个容器化的 Fedora 就可以用；`bash-color-prompt` 和 `bash-completion` 方便执行命令；`pciutils` 用于检查 PCIe 信息，比如执行 `sudo lspci -vvv` 检查 SSD 的连接速率是不是 2.5GT x1。

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
  -e HUB_URL="https://monitor.example.com" \
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
  -e MAX_PLAYERS=10 \
  -e INIT_MEMORY="512M" \
  -e MAX_MEMORY="6G" \
  -e VERSION=1.21.1 \
  -e TYPE="NEOFORGE" \
  -e MODS="/mods" \
  -e DIFFICULTY="hard" \
  -e MODE="survival" \
  -e VIEW_DISTANCE=16 \
  -e MAX_BUILD_HEIGHT=1024 \
  -e LEVEL_TYPE="minecraft:large_biomes" \
  -d docker.1panel.live/itzg/minecraft-server:java21-graalvm

```

我的树莓派 5 有 **8G** 内存，如果你的内存只有 4G，建议调低一些参数。其中 `MAX_MEMORY` 不要大于内存的 `90%`。另外，据说有人测试过 `graalvm` 的 JVM 性能优化非常好，~~虽然我看不出来~~。

另外还有一些参数可以关闭正版验证，具体参考[容器作者的文档](https://docker-minecraft-server.readthedocs.io/en/latest/)。我比较喜欢自己挑选模组而不是用整合包，整合包玩家看文档配置。如果你跟我一样只玩特定几个模组，在服务器安装模组可以右键点击复制 CurseForge/Modrinth 下载链接，在 `minecraft/mods` 目录执行 `curl -O https://cdn.modrinth.com/data/ordsPcFz/versions/xxx/xxx-1.1.1.jar`

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

## 备份

**服务器需要定期备份**！备份的档案应当放在额外的硬盘而非系统盘中。树莓派 5 自带 TF 卡槽，此处使用 TF 卡进行备份。在使用 TF 卡进行下面的操作前，请确保卡中没有重要的数据。

在树莓派上执行 `lsblk -f` 查看现有的硬盘，TF卡一般是 `mmcblk0`。如果有分区和文件系统存在，直接挂载使用也可以。我更倾向于格式化成 F2FS 或 XFS 文件系统来使用。由于 AlmaLinux 默认不包含格式化 F2FS 文件系统的工具，所以下面的操作将格式化原有 F2FS 文件系统成 XFS 文件系统。

```txt
almalinux@almalinux:~$ 
almalinux@almalinux:~$ 
almalinux@almalinux:~$
almalinux@almalinux:~$ lsblk -f
NAME FSTYPE FSVER LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
mmcblk0
│                                                                            
└─mmcblk0p1
     f2fs   1.16  sdcard 69260805-7aec-4022-b047-bc40e8379567     14G     6% /mnt
nvme0n1
│                                                                            
├─nvme0n1p1
│    vfat   FAT16 CIDATA B5DB-43D2                             364.5M    24% /boot
└─nvme0n1p2
     ext4   1.0   _/     0a52f693-6076-41b1-953a-2d86321e7d80  110.9G     5% /


almalinux@almalinux:~$ # 注意
almalinux@almalinux:~$ # 重新创建分区前先备份数据
almalinux@almalinux:~$ # 如果已经挂载请先取消挂载
almalinux@almalinux:~$ sudo fdisk /dev/mmcblk0

Welcome to fdisk (util-linux 2.40.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Command (m for help): m

Help:

  GPT
   M   enter protective/hybrid MBR

  Generic
   d   delete a partition
   F   list free unpartitioned space
   l   list known partition types
   n   add a new partition
   p   print the partition table
   t   change a partition type
   v   verify the partition table
   i   print information about a partition
   e   resize a partition

  Misc
   m   print this menu
   x   extra functionality (experts only)

  Script
   I   load disk layout from sfdisk script file
   O   dump disk layout to sfdisk script file

  Save & Exit
   w   write table to disk and exit
   q   quit without saving changes

  Create a new label
   g   create a new empty GPT partition table
   G   create a new empty SGI (IRIX) partition table
   o   create a new empty MBR (DOS) partition table
   s   create a new empty Sun partition table


Command (m for help): g
Created a new GPT disklabel (GUID: 73138C09-3E64-4886-9D58-AAFC9A9BFE4B).

Command (m for help): n
Partition number (1-128, default 1): 
First sector (2048-31211486, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-31211486, default 31209471): 

Created a new partition 1 of type 'Linux filesystem' and of size 14.9 GiB.
Partition #1 contains a f2fs signature.

Do you want to remove the signature? [Y]es/[N]o: y

The signature will be removed by a write command.

Command (m for help): p
Disk /dev/mmcblk0: 14.88 GiB, 15980298240 bytes, 31211520 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 73138C09-3E64-4886-9D58-AAFC9A9BFE4B

Device         Start      End  Sectors  Size Type
/dev/mmcblk0p1  2048 31209471 31207424 14.9G Linux filesystem

Filesystem/RAID signature on partition 1 will be wiped.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.


almalinux@almalinux:~$ 
almalinux@almalinux:~$ 
almalinux@almalinux:~$ # 接下来创建 XFS 文件系统
almalinux@almalinux:~$ lsblk -f
NAME        FSTYPE FSVER LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
mmcblk0                                                                             
└─mmcblk0p1                                                                         
nvme0n1                                                                             
├─nvme0n1p1 vfat   FAT16 CIDATA B5DB-43D2                             364.5M    24% /boot
└─nvme0n1p2 ext4   1.0   _/     0a52f693-6076-41b1-953a-2d86321e7d80  110.3G     6% /


almalinux@almalinux:~$ 
almalinux@almalinux:~$ 
almalinux@almalinux:~$ 
almalinux@almalinux:~$ sudo mkfs.xfs /dev/mmcblk0p1 
meta-data=/dev/mmcblk0p1         isize=512    agcount=4, agsize=975232 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=0  
data     =                       bsize=4096   blocks=3900928, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=0
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
Discarding blocks...Done.


almalinux@almalinux:~$ 
almalinux@almalinux:~$ 
almalinux@almalinux:~$ # 挂载并设置可读写权限
almalinux@almalinux:~$ sudo mount /dev/mmcblk0p1 /mnt
almalinux@almalinux:~$ sudo chmod 777 /mnt
almalinux@almalinux:~$ lsblk -f
NAME        FSTYPE FSVER LABEL  UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
mmcblk0                                                                             
└─mmcblk0p1 xfs                 0ccab63d-ef08-48d2-9ed8-937b7e5a3009   14.5G     2% /mnt
nvme0n1                                                                             
├─nvme0n1p1 vfat   FAT16 CIDATA B5DB-43D2                             364.5M    24% /boot
└─nvme0n1p2 ext4   1.0   _/     0a52f693-6076-41b1-953a-2d86321e7d80  110.3G     6% /

```

我的 TF 卡只有 16G 存储空间，所以不打算设置 `crontab` 定时任务，只手动执行备份。如果有需要定时备份，请搜索`/etc/fstab` 开机挂载 TF 卡和 CronTab 定时任务教程 。

```txt
almalinux@almalinux:~$ 
almalinux@almalinux:~$ 
almalinux@almalinux:~$ # 备份前先停止服务器运行
almalinux@almalinux:~$ systemctl --user stop container-minecraft.service


almalinux@almalinux:~$ # 使用 sudo 执行压缩以保留文件原始所有权，否则解压后会遇到权限问题
almalinux@almalinux:~$ # -a 自动识别文件后缀的压缩格式；-c 表示压缩；-f 表示输出的压缩文件名称
almalinux@almalinux:~$ sudo tar -acf /mnt/minecraft-$(date +%Y%m%d).tar.zst minecraft/
almalinux@almalinux:~$ sha1sum /mnt/minecraft-$(date +%Y%m%d).tar.zst | tee /mnt/minecraft-$(date +%Y%m%d).tar.zst.sha1
61b560a903402e1e8642d8134d840f0be769f223  /mnt/minecraft-20251219.tar.zst
almalinux@almalinux:~$ ls -alh /mnt
total 1.6G
drwxrwxrwx.  2 root      root       152 Dec 19 07:23 .
dr-xr-xr-x. 19 root      root      4.0K Dec 14 05:27 ..
-rw-r--r--.  1 root      root      641M Dec 19 06:37 minecraft-20251216.tar.zst
-rw-r--r--.  1 almalinux almalinux   69 Dec 19 06:37 minecraft-20251216.tar.zst.sha1
-rw-r--r--.  1 root      root      903M Dec 19 07:22 minecraft-20251219.tar.zst
-rw-r--r--.  1 almalinux almalinux   74 Dec 19 07:23 minecraft-20251219.tar.zst.sha1

# 使用 tar -vtf /mnt/minecraft-$(date +%Y%m%d).tar.zst 查看压缩文件的内容
# 使用 sha1sum -c /mnt/minecraft-$(date +%Y%m%d).tar.zst.sha1 校验哈希值
# 使用 sudo tar -axf /mnt/minecraft-$(date +%Y%m%d).tar.zst -C /path/to/data 解压存档到特定目录
# 注意，sudo 解压会保留文件原始所有权，不推荐使用非特权用户解压。

almalinux@almalinux:~$ 
almalinux@almalinux:~$ # 恢复服务运行
almalinux@almalinux:~$ systemctl --user start container-minecraft.service 
almalinux@almalinux:~$ systemctl --user status container-minecraft.service 
● container-minecraft.service - Podman container-minecraft.service
     Loaded: loaded (/home/almalinux/.config/systemd/user/container-minecraft.service; enabled; preset: disabled)
     Active: active (running) since Fri 2025-12-19 07:46:05 UTC; 1s ago
 Invocation: 1326c6f502454534bb4490c617cf0062
       Docs: man:podman-generate-systemd(1)
    Process: 375036 ExecStart=/usr/bin/podman start minecraft (code=exited, status=0/SUCCESS)
   Main PID: 375083 (conmon)
      Tasks: 15 (limit: 50283)
        CPU: 187ms
     CGroup: /user.slice/user-1000.slice/user@1000.service/app.slice/container-minecraft.service
             ├─375066 rootlessport
             ├─375073 rootlessport-child
             └─375083 /usr/bin/conmon --api-version 1 -c 138d17ee404884023c2784531870472d41bc941079268cb25cd2299a893bdf60 -u 138d17ee404884>

```

## Mods

以下是我家服务器和客户端安装的模组列表，这里的模组在服务器和客户端都必须安装。

- Carry on（2.2.2.11）
- Adorable Hamster Pets（3.4.2）
    - Kotlin for Forge（5.10.0）
    - Geckolib（4.8.2）
    - Fzzy Config（0.7.4）
    - Architectury API（13.0.8）
    - Patchouli（92）
- Autumnity（6.0.1）
    - Blueprint / Abnormals Core（8.0.8）
- All Tutta's Needs / Tutta's Doors（1.5.2）
- Another Furniture（4.0.1）
- Create（6.0.8）
    - Create: Diesel Generators（1.3.8）
    - Create: Connected（1.1.10）
    - Create Encased（1.7.2-fix2）
- Curios API（9.5.1）
- Farmer's Delight（1.2.9）
- Gravestone Mod（1.0.35）
- Immersive Aircraft（1.4.1）
- The Aether（1.5.10）
    - oωo（0.12.15.5）
- RoadWeaver（2.0.9）
- Sit（1.4）
- Storage Drawers（13.11.4）
- Touhou Little Maid（1.4.6）
- Traveler's Backpack（10.1.29）
- The Twilight Forest（4.7.3196）
- Antique Atlas（8.0.1）
    - UnionLib（12.0.18）
- Biomes O' Plenty（21.1.0.13）
    - GlitchCore（2.1.0.0）
    - TerraBlender（4.1.0.8）
- Everything is Copper（2.4.2）
- Exp Ore（0.3）
- Modular Golems（3.1.7）
    - L2 Library（3.0.7）
    - Golem Dungeons（2.0.4）

### Server Only

这里的模组只在服务器安装。

- Dungeons and taverns（4.4.4）
- Dungeons and Taverns Stronghold Overhaul（2.1.f）
- Dungeons and Taverns Ancient City Overhaul（3.2.1）
- Geophilic（3.4.4）
- Subsurface（1.0.4）
- Structory（1.3.12）
- Structory: Towers（1.0.14）
- Incendium（5.4.4）
- Tectonic（3.0.17）
    - Lithostitched（1.5.2）
- Yggdrasil（5.2.0）
- Dungeon Crawl（2.3.15）
- Epic Structures: Witch Huts（1.2.0）
- Mo' Structures（1.6.0）
- Villages&Pillages（1.0.3）
    - YUNG's API（5.1.6）
- Sunken Spires（1.0）
- Lios Seafaring Dungeons（0.0.5）

### Client Only

这里的模组只在客户端安装。

- Apple Skin（3.0.7）
- Clean Swing（1.9）
- Iris（1.8.12）
    - Sodium（0.6.13）
- Just Enough Items（19.25.1.332）
- I18nUpdateMod（3.7.0）
- Jade（15.10.3）
- Distant Horizons（2.4.1-b）

## 相关参考链接

- [Fixing 'Permission denied' Errors When Mounting Volumes in Rootless Podman](https://libraibex.com/posts/podman-volumes-permission-denied/)

