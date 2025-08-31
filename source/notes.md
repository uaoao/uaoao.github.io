---
title: 备忘录
type: notes
cover: img/notes-cover.jpg
---

# Linux 配置

## 禁用笔记本合盖休眠

1. 编辑 `/etc/systemd/logind.conf` 文件，如下：

```txt
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore

```

> 如果是 OSTree 系统，则需从 `/usr/lib/systemd/logind.conf` 复制一份到 `/etc/systemd/logind.conf` 再编辑。

2. 编辑 `/etc/UPower/UPower.conf` 文件，如下：

```txt
IgnoreLid=true

```

## GNOME 多显示器登陆配置

```bash
sudo cp $HOME/.config/monitors.xml /var/lib/gdm/.config/monitors.xml

```

## GNOME 桌面配置 ThinkPad USB 键盘

1. 安装【dmitry.goloshubov】作者的 GNOME 扩展。[下载链接](https://extensions.gnome.org/extension/3939/fnlock-switch-thinkpad-compact-usb-keyboard/)
2. 创建 udev 规则文件 `/usr/local/lib/udev/rules.d/61-thinkpad-keyboard.rules`，添加以下内容：

```txt
SUBSYSTEM=="input", DRIVERS=="lenovo", RUN += "/bin/sh -c 'FILE=$(find /sys/devices/ -name fn_lock 2>/dev/null); test -f $FILE && chmod 666 $FILE && ln -f -s $FILE /dev/fnlock-switch'"

```

3. 执行 `sudo udevadm control --reload && sudo udevadm trigger` 加载配置文件。

4. 默认配置下使用【Ctrl-Esc】可切换 Fn 锁定。也可以执行以下代码设置其他快捷键，但不能用【Fn】键代替：

```bash
cat << EOF | dconf load /org/gnome/shell/extensions/fnlock/
[/]
keybinding=['<Control>Escape']
EOF

```

## 笔记本配置内置键盘与 ThinkPad USB 键盘切换

1. 使用 `sudo libinput list-devices` 获取内置键盘【AT Translated Set 2 keyboard】的 `Kernel` 设备路径，例如 `/dev/input/event3`
2. 执行`sudo udevadm monitor --environment --udev`，然后插入外接 ThinkPad USB 键盘，终端将立即显示插拔事件的变量。获取外接键盘的 `ID_MODEL`变量，例如 `ID_MODEL=ThinkPad_Compact_USB_Keyboard_with_TrackPoint`
3. 创建 udev 规则文件 `/usr/local/lib/udev/rules.d/62-keyboard-switch.rules`，添加以下内容：（**注意部分内容修改成你系统中显示的结果**）

```txt
ACTION=="add", ENV{ID_MODEL}=="ThinkPad_Compact_USB_Keyboard_with_TrackPoint", RUN+="/bin/sh -c 'udevadm trigger --action=remove /dev/input/event3'"

ACTION=="remove", ENV{ID_MODEL}=="ThinkPad_Compact_USB_Keyboard_with_TrackPoint", RUN+="/bin/sh -c 'udevadm trigger --action=add /dev/input/event3'"

```

4. 执行 `sudo udevadm control --reload && sudo udevadm trigger` 加载配置文件。

- [参考链接](https://www.linuxquestions.org/questions/linux-desktop-74/udev-not-doing-remove-rules-841733/#post4146764)

# Bash 脚本

## vi

```bash
#!/usr/bin/sh

# run vim if:
# - 'vi' command is used and 'vim' binary is available
# - 'vim' command is used
# NOTE: Set up a local alias if you want vim -> vi functionality. We will not
# do it globally, because it messes up with available startup options (see
# ':help starting', 'vi' is not capable of '-d'). The introducing an environment
# variable, which an user must set to get the feature, will do the same trick
# as setting an alias (needs user input, does not work with sudo), so it is left
# on user whether he decides to use an alias:
#
# alias vim=vi
#
# in bashrc file.

if test -f /usr/bin/nvim; then
  exec /usr/bin/nvim "$@"
elif test -f /usr/bin/vim; then
  exec /usr/bin/vim "$@"
elif test -f /usr/libexec/vi; then
  exec /usr/libexec/vi "$@"
elif test -f /usr/bin/vi; then
  exec /usr/bin/vi "$@"
else
  test -f /usr/bin/nano
  exec /usr/bin/nano "$@"
fi

```

## pubip

```bash
#!/usr/bin/bash

/usr/bin/curl 4.ipw.cn &&\
echo "" &&\
/usr/bin/curl 6.ipw.cn &&\
echo ""

```

## now

```bash
#!/usr/bin/bash

echo -e "Date: $(/usr/bin/date +%Y/%m/%d)\nTime: $(/usr/bin/date +%H:%M:%S)"

```

## ll

```bash
#!/usr/bin/bash

/usr/bin/ls -alhF --color='auto' "$@"

```

## set-permission.sh

```bash
#!/usr/bin/bash
#
# Set permissions for all subdirectories
# and files in a directory.
#
# Arguments: <dir>
#
# File: 644
# Dir: 755
#
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter passed in is not a directory.
# [3] Error
#

# File permissions
fcode=644

# Directory permissions
dcode=755

function permission() {
    for file in "$1"/.* "$1"/*; do
        if [ -d "$file" ]; then
            test -L "$file" && continue || echo "[ ] $file"
            chmod $dcode "$file"
            permission "$file"
        elif [ -e "$file" ]; then
            echo "[-] $file"
            chmod $fcode "$file"
        fi
    done
}

if [ $# -ne 1 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <dir>"
    exit 1
elif [ ! -d "$1" ]; then
    echo "'$1' is not a directory!"
    echo "Usage: $(basename $0) <dir>"
    exit 2
fi

cd "$1" && permission "."

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $1 ]\033[0m \033[33m[ DONE ]\033[0m"
else
    echo -e "\n\033[32m[ $1 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 3
fi

```

## same-hash.sh

```bash
#!/usr/bin/bash
#
# find same hash
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter is not a hash list file.
# [3] Error.

if [ $# -ne 1 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <file>"
    exit 1
elif [ ! -f "$1" ]; then
    echo "'$1' is not a hash list file!"
    echo "Usage: $(basename $0) <file>"
    exit 2
fi

echo -e '\033[31m################# COMPARING #################\033[0m'
/usr/bin/python3 - "$1" << EOF
#########################################################
import sys

hash_dict = {}

with open(sys.argv[1], 'r', encoding="utf-8") as file:
    for line in file:
        hash_value = line.split(' ')[0].strip('\\\')
        file_name = ' '.join(line.split(' ')[1: ])

        exist = hash_dict.get(hash_value)
        if not exist:
            hash_dict[hash_value] = [file_name, ]
        else:
            hash_dict[hash_value] += [file_name, ]

index = 1
for key, values in hash_dict.items():
    if len(values) == 1:
        continue

    print(f"[{index}]\t {key}")
    for file_name in values:
        print(f"\t{file_name}", end='')

    print()
    index += 1
#########################################################
EOF
echo -e '\033[31m#################### DONE ###################\033[0m'

```

## same-file.sh

```bash
#!/usr/bin/bash
#
# Lists files with the
# same hash value in a directory.
#
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter is not a directory or file.
# [3] Error.
#

function check() {
    for file in "$1"/.* "$1"/*; do
        if [ -d "$file" ]; then
            test -L "$file" && continue 
            check "$file"
        elif [ -e "$file" ]; then
            sha1sum "$file"
        fi
    done
}

if [ $# -ne 1 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <dir>"
    exit 1
elif [ ! -d "$1" ]; then
    echo "'$1' is not a directory!"
    echo "Usage: $(basename $0) <dir>"
    exit 2
fi

result="/tmp/list-$(date +%Y%m%d%H%M%S).sha1"
cd "$1" && check "." | tee "$result"

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $1 ]\033[0m \033[33m[ "$result" ]\033[0m"
else
    echo -e "\n\033[32m[ $1 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 3
fi

echo -e '\033[31m################# COMPARING #################\033[0m'
/usr/bin/python3 - "$result" << EOF
#########################################################
import sys

hash_dict = {}

with open(sys.argv[1], 'r', encoding="utf-8") as file:
    for line in file:
        hash_value = line.split(' ')[0].strip('\\\')
        file_name = ' '.join(line.split(' ')[1: ])

        exist = hash_dict.get(hash_value)
        if not exist:
            hash_dict[hash_value] = [file_name, ]
        else:
            hash_dict[hash_value] += [file_name, ]

index = 1
for key, values in hash_dict.items():
    if len(values) == 1:
        continue

    print(f"[{index}]\t {key}")
    for file_name in values:
        print(f"\t{file_name}", end='')

    print()
    index += 1
#########################################################
EOF
echo -e '\033[31m#################### DONE ###################\033[0m'

```

## hash-mk.sh

```bash
#!/usr/bin/bash
#
# Generate a hash table for a directory.
#
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter passed in is not a directory.
# [3] The file exist.
# [4] Error.
#

function check() {
    for file in "$1"/.* "$1"/*; do
        if [ -d "$file" ]; then
            test -L "$file" && continue
            check "$file"
        elif [ -e "$file" ]; then
            sha1sum "$file"
        fi
    done
}

if [ $# -ne 2 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <src_dir> <target_file>"
    exit 1
elif [ ! -d "$1" ]; then
    echo "'$1' is not a directory!"
    echo "Usage: $(basename $0) <src_dir> <target_file>"
    exit 2
elif [ -e "$2" ]; then
    echo "'$2' exist!"
    echo "Usage: $(basename $0) <src_dir> <target_file>"
    exit 3
fi

result="$2"
cd "$1" && check "." | tee "$result"

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $1 ]\033[0m \033[33m[ "$result" ]\033[0m"
else
    echo -e "\n\033[32m[ $1 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 4
fi

```

## hash-cmp.sh

```bash
#!/usr/bin/bash
#
# Compare the hash table of a directory
# with the hash table file.
#
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter is not a directory.
# [3] The parameter is not a file.
# [4] Error.
#

function check() {
    for file in "$1"/.* "$1"/*; do
        if [ -d "$file" ]; then
            test -L "$file" && continue
            check "$file"
        elif [ -e "$file" ]; then
            sha1sum "$file"
        fi
    done
}

if [ $# -ne 2 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <dir> <file>"
    exit 1
elif [ ! -d "$1" ]; then
    echo "'$1' is not a directory!"
    echo "Usage: $(basename $0) <dir> <file>"
    exit 2
elif [ ! -f "$2" ]; then
    echo "'$2' is not a file!"
    echo "Usage: $(basename $0) <dir> <file>"
    exit 3
fi

result="/tmp/list-$(date +%Y%m%d%H%M%S).sha1"
cd "$1" && check "." | tee "$result" && cd - > /dev/null

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $1 ]\033[0m \033[33m[ "$result" ]\033[0m"
    echo -e '\033[31m################# COMPARING #################\033[0m'
    diff "$result" "$2"
    echo -e '\033[31m#################### DONE ###################\033[0m'
else
    echo -e "\n\033[32m[ $1 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 4
fi

```

## dirs-diff.sh

```bash
#!/usr/bin/bash
#
# Compare all normal files for both directories.
#
#
# exit code:
# [1] The number of parameters does not match.
# [2] The parameter passed in is not a directory.
#

function check() {
    for file in "$1"/.* "$1"/*; do
        if [ -d "$file" ]; then
            test -L "$file" && continue
            check "$file"
        elif [ -e "$file" ]; then
            sha1sum "$file"
        fi
    done
}

if [ $# -ne 2 ]; then
    echo "The number of parameters does not match!"
    echo "Usage: $(basename $0) <dir1> <dir2>"
    exit 1
elif [ ! -d "$1" ]; then
    echo "'$1' is not a directory!"
    echo "Usage: $(basename $0) <dir1> <dir2>"
    exit 2
elif [ ! -d "$2" ]; then
    echo "'$2' is not a directory!"
    echo "Usage: $(basename $0) <dir1> <dir2>"
    exit 2
fi

result_1="/tmp/list1-$(date +%Y%m%d%H%M%S).sha1"
cd "$1" && check "." | tee "$result_1" && cd - > /dev/null

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $1 ]\033[0m \033[33m[ "$result_1" ]\033[0m"
else
    echo -e "\n\033[32m[ $1 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 4
fi

echo '' && sleep 1s

result_2="/tmp/list2-$(date +%Y%m%d%H%M%S).sha1"
cd "$2" && check "." | tee "$result_2" && cd - > /dev/null

if [ $? -eq 0 ]; then
    echo -e "\n\033[32m[ $2 ]\033[0m \033[33m[ "$result_2" ]\033[0m"
    echo -e '\033[31m################# COMPARING #################\033[0m'
    diff "$result_1" "$result_2"
    echo -e '\033[31m#################### DONE ###################\033[0m'
else
    echo -e "\n\033[32m[ $2 ]\033[0m \033[31m[ ERROR ]\033[0m"
    exit 4
fi

```

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

# Android App 配置

- [FDroid Repository List](https://github.com/userkilled/FDroid-List-Repository)
- [rime 輸入方案和配置列表](https://github.com/ayaka14732/awesome-rime)

- 修复网络图标感叹号：

```bash
adb shell "settings put global captive_portal_http_url http://connect.rom.miui.com/generate_204"
adb shell "settings put global captive_portal_https_url https://connect.rom.miui.com/generate_204"

```

## 开源自由 App 订阅列表

### 官方构建

- [KernelSU Next](https://github.com/KernelSU-Next/KernelSU-Next)
- [MMRL](https://github.com/MMRLApp/MMRL)
- [Obtainium](https://github.com/ImranR98/Obtainium)
- [WebUIX](https://github.com/MMRLApp/WebUI-X-Portable)
- [阅读](https://github.com/gedoor/legado)
- [魔曰](https://github.com/SheepChef/Abracadabra)

### F-Droid 获取

- 【App Manager】**ROOT**
- 【AnkiDroid】
- 【AdAway】**ROOT**
- 【Droid-ify】 F-Droid 第三方客户端
- 【Feeder】
- 【Fork Client (Forkgram)】Telegram 第三方客户端
- 【LocalSend】
- 【Moshidon】 Mastdon 第三方客户端
- 【Nextcloud】
- 【Open Note】
- 【PiliPala】Bili 第三方客户端
- 【Shelter】工作空间
- 【sing-box】
- 【Breezy Weather】
- 【Termux】
- 【Unciv】类《文明5》开源游戏
- 【VLC】
- 【Loop Habit Tracker】 习惯记录、提醒
- 【ZipXtract】解压工具
- 【Barcode Scanner】条码和二维码扫描/生成条码和二维码

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

# NextCloud

## 精选插件

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

