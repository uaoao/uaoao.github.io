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

# Openwrt

## 软件安装记录

### 基本软件包

```txt
base-files ca-bundle dropbear firewall4 fitblk fstools kmod-crypto-hw-safexcel kmod-gpio-button-hotplug kmod-leds-gpio kmod-nft-offload kmod-phy-aquantia libc libgcc libustream-mbedtls logd mtd netifd nftables odhcp6c odhcpd-ipv6only opkg ppp ppp-mod-pppoe procd-ujail uboot-envtools uci uclient-fetch urandom-seed urngd wpad-basic-mbedtls kmod-hwmon-pwmfan kmod-i2c-mux-pca954x kmod-eeprom-at24 kmod-mt7996-firmware kmod-mt7996-233-firmware kmod-rtc-pcf8563 kmod-sfp kmod-usb3 e2fsprogs f2fsck mkf2fs mt7988-wo-firmware luci

dnsmasq-full kmod-nvme kmod-fs-f2fs lsblk block-mount gdisk pciutils
```

### 常见文件系统

```txt
kmod-fs-btrfs kmod-fs-exfat kmod-fs-ext4 kmod-fs-vfat kmod-fs-xfs kmod-fs-squashfs kmod-fs-ntfs3
```

### 其他 TODO

- luci-app-cloudflared
- ddns-scripts-cloudflare (require curl ca-bundle)
- luci-app-ddns (require bind-host)
- bind-host
/etc/ssl/certs/ca-certificates.crt


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
      MOTD: "Welcome to mc6.wagada.cc"
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

