---
title: OpenWrt配置
type: notes
cover: img/notes.webp
---

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
user_script_ver="v1.2"

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
    if [[ $user_net_loop == "4" ]]
    then
        echo -e "[$(date +%Y/%m/%d) $(date +%H:%M:%S)] ERROR:\tCanot resume network over 4 times!" >> $user_log_path
        echo -e "[$(date +%Y/%m/%d) $(date +%H:%M:%S)] ERROR:\tIgnore network checking!" >> $user_log_path
        user_net_loop=-1
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

if [[ $user_net_loop != "-1" ]]
then
    echo -e "[$(date +%Y/%m/%d) $(date +%H:%M:%S)] INFO:\tNetwork check finished!" >> $user_log_path
fi

echo >> $user_log_path
unset user_net_loop


###### 此处添加自定义内容  #########


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

先从 [feed.sh](https://github.com/nikkinikki-org/OpenWrt-nikki/blob/main/feed.sh) 中复制脚本内容，在 OpenWrt 中保存成文件并执行，然后安装软件。

> 可能与【DNS Over Https】功能相冲突。

```txt
luci-app-nikki

```

### NUT (APC-UPS)

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

### SFTP

让【Dropbear】支持包括 `scp` 以及 mDNS 服务宣告功能的 SFTP 服务。无需额外设置，即装即用。

```txt
openssh-sftp-avahi-service
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
curl
tree
zstd
gzip
ca-certificates
vim-fuller
zoneinfo-all
diffutils

f2fs-tools
dosfstools
exfat-mkfs
squashfs-tools-mksquashfs
swap-utils
xfs-mkfs

coreutils-md5sum
coreutils-sha1sum
coreutils-sha256sum
coreutils-sha512sum

```

