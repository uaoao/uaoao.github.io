---
title: OpenWrt配置NUT管理APC-UPS
date: 2025-6-13
tags:
  - 2025年
  - OpenWrt
  - Linux
  - APC-UPS
  - NUT
  - 配置
  - 教程
---


## 前言

早在大学读书玩Bittorrent的时候，我就购买了一台APC不间断电源。以前经历过几次学校或家里突然停电的情况。那时候我用的只有一台笔记本电脑和内置的硬盘，所以即使断电也不会就突然关机。自从开始用外接机械硬盘玩BT下载，我开始对突如其来的断电而担心受怕。即使在玩BT的时期到现在，都没有遇到突然断电导致数据损毁，为了以防万一，花五百多买了一台带远程控制功能的 **APC BK650M2-CH**

最近在折腾OpenWrt上跑各种服务，也打算后续外接一块便携式机械硬盘，因此很有必要把断电自动关机的功能弄好。在此之上，局域网内如果后续有打算添加7*24小时不关机的服务器设备，也可从中获益，只需安装Client接受开关机信号即可。


## 前提条件

- [APC BK650M2-CH](https://networkupstools.org/ddl/APC/Back-UPS_BK650M2-CH.html)
- 安装了 OpenWrt 的硬件必须有 USB 接口，同时 UPS 与需要安装 NUT 的设备相连。


## 安装所需软件包

- `luci-app-nut` 提供 LuCI 界面和 NUT 的大部分功能
- `usbutils` 提供管理USB设备的命令
- `nut-driver-usbhid-ups` 设备 APC BK650M2-CH 使用的驱动。其他型号的可以在[OpenWrt官方文档上找到](https://openwrt.org/docs/guide-user/services/ups/software.nut)。

其中，`luci-app-nut`主要包括以下三个部分：

- `nut-server`：对应 NUT-Server 选项卡。与 UPS 直接连接，利用驱动程序提供UPS管理服务。
- `nut-upsmon`：对应 NUT-Monitor 选项卡。监控 UPS 服务状态，触发关机事件。
- `nut-web-cgi`：对应 NUT-CGI 选项卡。提供图形化网页管理界面，可以查看 UPS 状态，以及修改阈值。

其他相关软件包可以在[OpenWrt官网](https://openwrt.org/docs/guide-user/services/ups/software.nut#package_selection)找到。


## 服务配置

以下是必填的配置，其他配置按需求填写，但是建议不要随意填写 NUT Server 选项卡中 Driver Configuration 这部分的内容，否则服务无法正常启动。我在这里吃了不少亏。看网上教程博主们填了的部分我也跟着填，结果服务无法启动，排查问题费了很多时间。一些问题可以参考文章底部的链接。

### NUT Server 选项卡

- NUT User
  - Username: `monuser`（填自己想填的）
  - Password: `secret`（填自己想填的）
  - Allowed actions: `Set variables`，`Forced Shutdown`
  - Role: `Primary`（Primary 与 Auxiliary 分别是 **主** 与 **从** 的角色。权限不同，一般直连 UPS 的设备为主，其他设备为从）

- Addresses on which to listen
  - IP Address: `0.0.0.0`（对局域网开放）
  - Port: `3493`

- Driver Configuration: `apcups`（填自己想填的）
  - Driver: `usbhid-ups`
  - Port: `auto`
  - Serial Number: `9B2323A06928`（填自己设备的，用 `lsusb -v` 即可获得）

### NUT Monitor 选项卡

- UPS Primary
  - Name of UPS: `apcups`
  - Hostname or address of UPS: `127.0.0.1`
  - Port: `3493`
  - Power value: `1`
  - Username: `monuser`
  - Password: `secret`

### NUT CGI 选项卡

- Host
  - UPS name: `apcups`
  - Hostname or IP address: `127.0.0.1`
  - Port: `3493`
  - Display name: `American Power Conversion Back-UPS BK650M2-CH`

- Control UPS via CGI: `true`


## 检查服务状态

1. 保存并应用以上的配置后，使用 SSH 登陆 OpenWrt。
2. 执行命令 `/etc/init.d/nut-server restart`。
3. 执行命令 `/etc/init.d/nut-server status`。

如果显示`running`则说明服务正常执行中；但如果显示 `running (1/2)` 则说明 NUT Server 选项卡的 Driver Configuration 部分没有正确地设置。可以通过执行 `/etc/init.d/nut-server info` 来查看具体哪个部分失败了。比如 `apcups` 部分的 `running` 值是 `false`，说明执行失败。而它实际执行的命令是 `/lib/nut/usbhid-ups -D -a apcups -u nut`。

如果你用的UPS驱动跟我一样，则可以尝试手动执行这条命令，以查看结果。

```bash
root@OpenWrt:~# /lib/nut/usbhid-ups -D -a apcups -u nut
Network UPS Tools - Generic HID driver 0.52 (2.8.1)
USB communication driver (libusb 1.0) 0.46
   0.000000	[D1] Built-in default or configured user for drivers 'root' was ignored due to 'nut' specified on command line
   0.000040	[D1] Network UPS Tools version 2.8.1 (release/snapshot of 2.8.1) built with aarch64-openwrt-linux-musl-gcc (OpenWrt GCC 13.3.0 r28698-0db2af95bc) 13.3.0 and configured with flags: --target=aarch64-openwrt-linux --host=aarch64-openwrt-linux --build=x86_64-pc-linux-gnu --disable-dependency-tracking --program-prefix= --program-suffix= --prefix=/usr --exec-prefix=/usr --bindir=/usr/bin --sbindir=/usr/sbin --libexecdir=/usr/lib --sysconfdir=/etc --datadir=/usr/share --localstatedir=/var --mandir=/usr/man --infodir=/usr/info --disable-nls --sysconfdir=/etc/nut --datadir=/usr/share/nut --with-dev --with-usb --without-avahi --without-snmp --with-serial --without-doc --with-neon --without-powerman --without-wrap --with-hotplug-dir=/etc/hotplug --with-cgi --without-ipmi --without-freeipmi --without-linux-i2c --without-ssl --without-libltdl --without-macosx_ups --with-statepath=/var/run/nut --with-pidpath=/var/run --with-drvpath=/lib/nut --with-user=root --with-group=root --with-gd-includes='-I/builder/shared-workdir/build/sdk/staging_dir/target-aarch64_cortex-a53_musl/usr/include -I/builder/shared-workdir/build/sdk/staging_dir/target-aarch64_cortex-a53_musl/usr/include/freetype2 -I/builder/shared-workdir/build/sdk/staging_dir/target-aarch64_cortex-a53_musl/usr/include/libpng16' --with-gd-libs='-L/builder/shared-workdir/build/sdk/staging_dir/target-aarch64_cortex-a53_musl/usr/lib -lgd'
   0.000097	[D1] debug level is '1'
   0.000347	[D1] Succeeded to become_user(nut): now UID=113 GID=113
   0.000411	[D1] upsdrv_initups (non-SHUT)...
   0.003543	Can't claim USB device [051d:0002]@0/0: Entity not found
   0.003599	upsnotify: failed to notify about state 4: no notification tech defined, will not spam more about it
```

我这里的问题就是在配置部分填入了【USB Product Id】和【USB Vendor Id】。查了很久的资料才知道 NUT 驱动通过识别设备序列号来查找设备，填了相关 ID 或名称会有各种问题。可能其他博主的 NUT 驱动版本是通过识别 ID 来查找设备。

4. 执行 `upsc apcups`，也可以查看设备的各种信息，而且内容更详细具体。
5. 打开OpenWrt 的管理网页，来到 Service —— Network UPS Tools —— NUT CGI 选项卡，点击标题下面的【Go to NUT CGI】，点击【Statistics】，可以看到【Status】是 `ONLINE`。
6. 点击 UPS 的名称，可以进入可视化界面查看 UPS 信息。
7. 在【Settings】部分可以修改 UPS 阈值。
8. **如果有时间，建议尝试将 UPS 断电测试是否真的能提前关机**。


## 其他管理 APC-UPS 的方案

我以前没有接触 NUT 的时候，用的【apcupsd】方案。这个软件包也能实现断电提前关机，只是配置和功能仅针对直连的主机，其他设备无法共享 UPS提前关机。主要是这个方案配置起来非常简单，安装软件后修改一个配置文件（有的系统不用修改）就能使用了。


## 相关参考链接

- [Can't claim USB device 【051d:0003】@0/0: Entity not found](https://community.home-assistant.io/t/nut-tools-only-1-usb-ups-at-a-time-works-solved/765244/7)
- [OpenWRT软路由UPS教程-基于山特TG-BOX 850](https://www.luqijian.com/2023/01/08/openwrt-with-ups-tutorial-based-on-santak-tg-box-850/)
- [APC UPS通过NUT转换网络UPS](https://blog.12ms.xyz/archives/2250)
- [PVE/Linux安装nut管理apc BK650M2-CH ups自动关机](https://www.haiyun.me/archives/1425.html)
- [一鱼多吃，通过NUTserver让多台机器同时使用同一个UPS：NUT主服务器配置篇](https://www.bilibili.com/opus/918271086007681026)
- [APC BackUps ES-500 - Linksys EA3500 - LuCI graphs](https://openwrt.org/docs/guide-user/services/ups/apcupsd_es500)
- [使用NUT解决BK650M2-CH失联问题（一）](https://www.alainlam.cn/?p=56)

