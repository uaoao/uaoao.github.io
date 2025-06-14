---
title: 备忘录
type: notes
cover: img/notes-cover.jpg
---

# Linux 配置

## Linux 禁用笔记本合盖休眠

1. 编辑 `/etc/systemd/logind.conf` 文件，如下：

```txt
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore

```

2. 编辑 `/etc/UPower/UPower.conf` 文件，如下：

```txt
IgnoreLid=true

```

# Bash 脚本

## netdisk.sh

```bash
#!/usr/bin/bash

HOST=ssh.server
USER=username
KEY_FILE=$HOME/.ssh/user.key
PORT=22

REMOTE_PATH=/home
LOCAL_PATH=$HOME/NetDisk

install -dm755 $LOCAL_PATH
/usr/bin/sshfs -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3,IdentityFile="$KEY_FILE",idmap=user,uid=$(id -u),gid=$(id -g),default_permissions -C -p $PORT ${USER}@${HOST}:$REMOTE_PATH $LOCAL_PATH

```

## tbox.sh

```bash
#!/usr/bin/bash

install -dm700 $HOME/TboxHome
/usr/bin/toolbox run env HOME=$HOME/TboxHome bash

```

## pyserverd.sh

```bash
#!/usr/bin/bash

install -dD $HOME/.cache $HOME/Public
nohup /usr/bin/python3 -m http.server 8888 -d $HOME/Public >>$HOME/.cache/pyserver.log 2>&1 &

```

## mount_hdd.sh

```bash
#!/usr/bin/bash
#
if [ -e /dev/sda1 ]; then
	sudo mount -r /dev/sda1 /home/pi.public
else
	echo "ERROR: HDD does't exist!"
fi

```

## add_netdisk_user.sh

```bash
#!/usr/bin/bash
#需配置禁用root登陆
#需配置禁用su能力
#需配置禁用空密码登陆

read -p "Enter new user account: " new_account
read -p "New account '$new_account', ok?[N/y]: " is_ok

if [ "$is_ok" == 'Y' -o "$is_ok" == 'y' ]; then
    home_dir="/home/${new_account}"
    useradd "$new_account" && mkdir -m 0700 "${home_dir}"

    if [ $? == 0 ]; then
        chown "$new_account":"$new_account" "${home_dir}"
        mkdir -m 0755 "${home_dir}.public"
        chown "$new_account":"$new_account" "${home_dir}.public"

        cd "${home_dir}"

        mkdir -m 0500 ".ssh"
        chown "$new_account":"$new_account" ".ssh"

        ssh-keygen -t rsa-sha2-256 -C "${new_account} for netdisk" \
            -f ".ssh/temp"

        mv .ssh/temp .ssh/"$new_account".key
        chown "$new_account":"$new_account" .ssh/"$new_account".key
        chmod 0400 .ssh/"$new_account".key

        mv .ssh/temp.pub .ssh/auth_keys
        chown "$new_account":"$new_account" .ssh/auth_keys
        chmod 0444 .ssh/auth_keys

        ls -alh .ssh/
    fi
fi

```

# OpenWrt

## OpenWrt 用户自定义启动脚本

```bash
# OpenWrt 自定义启动脚本
# 将脚本粘贴在以下位置：
# Web Luci -- System -- Startup -- Local Startup

###################################
######### 脚本变量设置 ############
###################################
# 自定义脚本执行日志存放位置
user_log_path=/tmp/boot_script.log
# 脚本版本号
user_script_ver="v1.0"

###################################
######### 脚本开始执行 ############
###################################
sleep 60s
echo -e "[$(date +%Y/%m/%d) $(date +%H:%M:%S)] INFO:\tUser boot script ${user_script_ver} start!" >> $user_log_path

###################################
######### 联网状态检查 ############
###################################
# 读取以2开头的公网IPv6地址
# 如果地址不存在，则尝试重启网络，并且再次尝试检查网络状态
# 依赖 ip 命令

echo -e "[$(date +%Y/%m/%d) $(date +%H:%M:%S)] INFO:\tNetwork check start!" >> $user_log_path
/etc/init.d/network status >> $user_log_path 2>&1
user_net_loop=0

while [[ "$(ip a | awk '/inet6 2/ {print $0}')" == "" ]]
do
    if [[ $user_net_loop == "3" ]]
    then
        echo -e "[$(date +%Y/%m/%d) $(date +%H:%M:%S)] ERROR:\tCanot resume network over 3 times!" >> $user_log_path
        break
    fi

    echo -e "[$(date +%Y/%m/%d) $(date +%H:%M:%S)] ERROR:\tNetwork Error!" >> $user_log_path
    echo -e "[$(date +%Y/%m/%d) $(date +%H:%M:%S)] WARNING:\tRestart network..." >> $user_log_path
    user_net_loop=$((${user_net_loop}+1))
    /etc/init.d/network restart >> $user_log_path 2>&1
    echo -e "[$(date +%Y/%m/%d) $(date +%H:%M:%S)] INFO:\tNetwork restart finishd!" >> $user_log_path
    sleep 60s
    /etc/init.d/network status >> $user_log_path 2>&1
done

echo -e "[$(date +%Y/%m/%d) $(date +%H:%M:%S)] INFO:\tNetwork check finishd!" >> $user_log_path
echo >> $user_log_path
unset user_net_loop


###### 此处编写自定义内容  #########


###################################
######### 脚本执行完成 ############
###################################
echo -e "[$(date +%Y/%m/%d) $(date +%H:%M:%S)] INFO:\tUser boot script ${user_script_ver} finishd!" >> $user_log_path
echo >> $user_log_path
unset user_log_path
unset user_script_ver
exit 0

```

## 软件安装记录

### 基本软件包

```txt
base-files ca-bundle dropbear firewall4 fitblk fstools kmod-crypto-hw-safexcel kmod-gpio-button-hotplug kmod-leds-gpio kmod-nft-offload kmod-phy-aquantia libc libgcc libustream-mbedtls logd mtd netifd nftables odhcp6c odhcpd-ipv6only opkg ppp ppp-mod-pppoe procd-ujail uboot-envtools uci uclient-fetch urandom-seed urngd wpad-basic-mbedtls kmod-hwmon-pwmfan kmod-i2c-mux-pca954x kmod-eeprom-at24 kmod-mt7996-firmware kmod-mt7996-233-firmware kmod-rtc-pcf8563 kmod-sfp kmod-usb3 e2fsprogs f2fsck mkf2fs mt7988-wo-firmware luci

dnsmasq-full kmod-nvme kmod-fs-f2fs lsblk block-mount gdisk pciutils

```

### 常用文件系统相关内核模块

```txt
kmod-fs-f2fs
kmod-fs-btrfs
kmod-fs-exfat
kmod-fs-ext4
kmod-fs-vfat
kmod-fs-xfs
kmod-fs-squashfs
kmod-fs-ntfs3
kmod-fs-isofs
kmod-fuse

```

### Cloudflare DDNS

```txt
ddns-scripts-cloudflare
luci-app-ddns
bind-host

```

### Cloudflare Zero Trust Tunnels

```txt
luci-app-cloudflared

```

### Docker

```txt
luci-app-dockerman

```

### DNS Over Https

```txt
luci-app-https-dns-proxy

```

### Nikki

- 先从 [feed.sh](https://github.com/nikkinikki-org/OpenWrt-nikki/blob/main/feed.sh) 中复制脚本内容，在 OpenWrt 中保存成文件并执行，然后安装软件。

> 可能与【DNS Over Https】功能相冲突。

```txt
luci-app-nikki

```

### NUT

```txt
luci-app-nut
usbutils
nut-driver-usbhid-ups

```

### Statistics

```txt
luci-app-statistics
collectd-mod-thermal

```

### WatchCat

定时 Ping，监控联网状态并触发重启。

```txt
luci-app-watchcat

```

### USB / NVME 相关内核模块

#### 基础驱动模块

```txt
kmod-nvme
kmod-usb2
kmod-usb3
kmod-usbmon         # USB 性能监视
kmod-usb-hid        # HID 输入设备
kmod-usb-wdm        # USB 无线设备
kmod-usb-net        # USB 转网卡
kmod-usb-serial     # USB 转串口
kmod-usb-storage    # USB 转储存

```

#### 硬件特定驱动模块

```txt
kmod-usb-storage-extras
kmod-usb-storage-uas
kmod-usb-serial-simple
kmod-usb-net-asix-ax88179  # USB HUB 网卡
kmod-usb-net-cdc-ncm       # USB HUB 网卡
kmod-usb-net-rndis         # 手机网络共享转 USB

```

### 杂项

```txt
lsblk
usbutils
pciutils
nvme-cli
gdisk fdisk
mount-utils
afuse
curl
tree
zstd
gzip
ca-certificates
vim-fuller
zoneinfo-all

f2fs-tools
dosfstools
exfat-mkfs
squashfs-tools-mksquashfs
swap-utils
xfs-mkfs

```

# Android App 配置

- [FDroid Repository List](https://github.com/userkilled/FDroid-List-Repository)
- [rime 輸入方案和配置列表](https://github.com/ayaka14732/awesome-rime)

- 修复网络图标感叹号：

```bash
adb shell "settings put global captive_portal_http_url http://connect.rom.miui.com/generate_204"
adb shell "settings put global captive_portal_https_url https://connect.rom.miui.com/generate_204"

```

# Docker Compose

## fedora server compose.yml

```yaml
services:
  fedora-docker:
    image: library/fedora
    network_mode: "host"
    volumes:
      - ./root:/root
      - ./home:/home
    tty: true
    stdin_open: true
    restart: unless-stopped

```

## minecraft server compose.yml

```yaml
services:
  minecraft:
    image: itzg/minecraft-server
    container_name: "MCServer"
    ports:
      - 25565:25565
    volumes:
      - ./minecraft-data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      SERVER_NAME: "MCServer"
      ICON: "https://www.minecraft.net/content/dam/minecraftnet/games/minecraft/logos/Homepage_Gameplay-Trailer_MC-OV-logo_300x300.png"
      VERSION: "LATEST"
      OVERRIDE_ICON: "FALSE"
      EULA: "TRUE"
      INIT_MEMORY: "512M"
      MAX_MEMORY: "4G"
      TYPE: "VANILLA"
      MOTD: "Welcome to Minecraft"
      DIFFICULTY: "hard"
      MAX_PLAYERS: 40
      FORCE_GAMEMODE: true
      MAX_BUILD_HEIGHT: 512
      MODE: "survival"
      SETUP_ONLY: false
      VIEW_DISTANCE: 10
      PVP: true
      LEVEL_TYPE: "minecraft:large_biomes"
      USE_AIKAR_FLAGS: false
      NETWORK_COMPRESSION_THRESHOLD: 512
      ENABLE_RCON: true
      RCON_PASSWORD: "1j9fR7_#9~Vs"
      RCON_PORT: 25575
    restart: unless-stopped

  rcon:
    image: itzg/rcon
    container_name: "RCON"
    depends_on:
      - minecraft
    ports:
      - "3000:4326"
      - "4327:4327"
    volumes:
      - ./rcon-data:/opt/rcon-web-admin/db
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    environment:
      RWA_ENV: "TRUE"
      RWA_ADMIN: "TRUE"
      RWA_PASSWORD: "Juf7%n.0pFg1"
      RWA_RCON_HOST: "MCServer"
      RWA_RCON_PASSWORD: "1j9fR7_#9~Vs"
      RWA_RCON_PORT: 25575
    restart: unless-stopped

```

# DNS

## AliDNS

```txt
https://dns.alidns.com/dns-query
223.6.6.6
223.5.5.5
2400:3200:baba::1
2400:3200::1
```
