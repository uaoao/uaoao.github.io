---
title: BananaPi-R4路由器开发板使用心得
date: 2025-4-16
tags:
  - BananaPi-R4(BPI-R4)
  - OpenWrt
  - 开发板
  - 路由器
  - 心得
  - Linux
  - 2025年
---

## 购买原因

由于公网IPv4的消失导致公网IPv6被迫普及，加上[智能光猫内置反诈程序监控用户上网行为](https://www.v2ex.com/t/986550)，于是今年早些时候购入了BananaPi-R4（BPI-R4）路由器开发板。目前这个开发板已经有最新的OpenWrt稳定版支持，所以即使是技术小白也能轻松的安装系统上手使用。

如果家中千兆以下宽带，且只是想光猫桥接路由器拨号的话，不建议购买这个开发板。因为如果要实现拨号+路由+WIFI这些基础功能，你需要购买：

- 支持的猫棒
- BPI-R4本体
- BE14无线网卡
- 金属外壳
- 散热片、散热硅脂、散热器
- WIFI天线
- 8G以上的TF卡
- ft232 USB串口线

加起来有一千多块钱，而且官方提供的Ubuntu、Debian、OpenWrt都属于BSP内核，需要有一定嵌入式系统开发经验的人使用。好在年初发布的OpenWrt24.10主线支持了这块板子，现在即使小白也能上手使用了。

我购买这块板子主要是想实现All In One，一台路由器实现拨号上网、路由、无线网络、家庭云存储、部署一些额外的服务等等。同时由于没有公网IPv4，市面上便宜的路由器一般不带有IPv6的防火墙设置功能，所以需要一台能够设置IPv6防火墙的路由器，加上双10G SFP接口、高扩展性的OpenWrt系统、4G内存+8G eMMC存储，以及PCIe3的M.2固态硬盘接口，这配置简直吊打市面上所有家用路由器。

## 电源与UPS

我用的是UGREEN 65W Type-C充电器，板子本身支持PD协议而且必须协商到20V才能供电。如果板子上要插网卡和固态硬盘，12V 3A的官方充电器供电完全不够，建议直接购买高质量的65W PD协议Type-C充电器。

我不知道板子是否支持双路供电，官方文档没写。如果要用UPS最好是充电器外接UPS，USB通信，这样比较安全稳定。

## 散热

如图所示是我的散热方案。最关键的部分，就是板子背面固态和无线网卡的2.5mm厚导热硅脂片+贴在金属外壳底部的150\*95\*15mm大功率散热片+12cm外壳散热风扇。

右侧网卡底部的硅脂片没有显示出来，除了网卡底部有硅脂片，网卡与主板之间的缝隙也含了一块硅脂片。导热硅脂片的具体宽高需要购买整块硅脂片（建议两片100\*50mm）后用小刀切成符合的大小，比较麻烦，这里就不再详细说明了。

![BPI-R4散热方案](images/BPI-R4散热方案.webp)

如果没有这套组合，长时间运行时内部温度会达到80摄氏度以上。尤其是固态硬盘和无线网卡，是造成高温的主要原因。加上这套组合后，内部温度可以稳定在40～50摄氏度左右。

有人可能会疑惑为什么不把底部的散热片和风扇放在顶部，当然是因为官方设计的外壳顶部不是平面。而且热源本身就在底部，底部也贴了硅脂，热传导效率显然更高，在底部散热效果也最明显。

还有，请注意底部风扇与底部散热片之间缝隙的距离不应小于1cm，风扇太近散热效果不好。

## OpenWrt 安装到TF卡

准备好ft232串口线、8G以上的TF卡+读卡器、互联网、电脑在同网段的局域网、网线。

1. 在[OpenWrt固件选择器](https://firmware-selector.openwrt.org/?version=24.10.1&target=mediatek%2Ffilogic&id=bananapi_bpi-r4)页面下载最新的稳定版`24.10.1`（截止撰稿时）的SDCARD.IMG.GZ压缩包。

2. 下载完成后使用以下命令测试文件是否完整：

```bash
sha256sum openwrt-24.10.1-mediatek-filogic-bananapi_bpi-r4-sdcard.img.gz
```

用肉眼大概对比一下计算结果与官网的哈希是否相同，如果不同就重新下载。然后解压出IMG文件：

```bash
gzip -dk openwrt-24.10.1-mediatek-filogic-bananapi_bpi-r4-sdcard.img.gz
```

3. 读卡器插入TF卡，在电脑上检查是哪个块设备。我这里显示的是`/dev/sde`，那么用`dd`命令直接把文件写入进去（注意TF卡里的数据先备份好，这个命令会擦除所有数据）：

```bash
sudo dd if=openwrt-24.10.1-mediatek-filogic-bananapi_bpi-r4-sdcard.img of=/dev/sde
```

4. 将ft232串口线接在BPI-R4主板USB3边上的3PIN针上：

- 红色线5V供电，不用插
- 黑色线GND，插板子的G针
- 绿色线TX，插板子的RX针
- 白色线RX，插板子的TX针

5. TF卡写入系统镜像文件后插在BPI-R4上，把板子侧边的切换开关SW3打在A1B1的位置，往上拨是0；往下拨是1。然后插电开机。

```txt
| SW3  |  A  |  B  |
| ---- | --- | --- |
| NAND |  0  |  1  |
| eMMC |  1  |  0  |
| SD   |  1  |  1  |
| Halt |  0  |  0  |
```

6. 安装使用命令`screen`来与板子通信，电脑终端会快速输出大量启动信息，然后启动输出内核日志。

```bash
sudo screen /dev/ttyUSB0 115200
```

7. 在板子的LAN口接上网线连接电脑，登陆 http://192.168.1.1 即可进入TF卡中的OpenWrt系统的管理网页。

## 猫棒的使用和设置

我家宽带是联通500M GPON，光猫自带一个千兆口和三个百兆口，但这不足以支撑多台电脑的带宽需求。

猫棒需要根据光猫使用的PON协议来购买，EPON与GPON不能混用。而且有些猫棒可能板子不支持，最好去 [Banana Pi的论坛](https://forum.banana-pi.org/) 问或查下你要买的猫棒是否支持。我用的是华为的MA5671A，卖家已经刷好了OpenWrt系统。

进运营商光猫Admin后台拿到光猫的SN、MAC、LOID、上网账号和密码等信息，然后将猫棒插在SFP LAN口，按照猫棒卖家的教程把信息填写进去就可以了，其他的配置不要动。猫棒配置好了重启再试试直接电脑拨号上网，能成功的话以后基本不需要再配置猫棒了，直接插SFP WAN口用就可以。（不建议将电脑长时间直接暴露在公网）（不建议直接插拔猫棒，请关机后插拔）

将猫棒插在SFP WAN口后，需要进入OpenWrt的网络接口配置界面删除默认的 DHCP WAN 口并添加 PPPoE 协议的新接口，如图所示选择 eth2 物理接口（我已经配置好了所以图中不同）。

![OpenWrt设置PPPoE](images/OpenWrt设置PPPoE.webp)

然后在 General Settings 中填写 PAP/CHAP 账号和密码，也就是你宽带的账号和密码。在Firewall Settings中防火墙区域选择WAN区域。保存重启路由器，LAN口连上电脑就可以正常上网了。

## 安装OpenWrt到板载SPI-NAND和eMMC

在操作之前，你应该已经知道了BPI-R4这块板子本身包含两个存储器。一个是128MB的SPI-NAND；另一个则是8GB的eMMC。当插入TF卡时，板载eMMC会被屏蔽，而SPI-NAND和TF卡作为块设备被识别到。当然，这其中不包括M.2插槽的固态硬盘和外接USB存储，这些外部设备目前只有Linux内核启动并载入相应模块后才能被识别，U-Boot启动状态目前无法识别。

1. 首先你需要将OpenWrt镜像刷入TF卡并插入BPI-R4，并且用ft232串口线连接板子和电脑。

2. 将SW3开关切换到SD模式，也就是A1B1位置，插电开机，电脑终端用`screen`命令进入系统。

3. 先让系统默认状态启动一遍，保证能正常开机进入OpenWrt网站。然后按板子上的RST重启按钮重启进入U-Boot的启动菜单，并快速按电脑键盘的上下键防止跳过，类似进入UEFI选项的感觉。内容大概是这样的：

```txt
        ( ( ( OpenWrt ) ) )  [SD card]       U-Boot 2024.10-OpenWrt-r28597-0425664679 (Apr 13 2025 - 16:38:32 +0000)

      1. Run default boot command.
      2. Boot system via TFTP.
      3. Boot production system from SD card.
      4. Boot recovery system from SD card.
      5. Load production system via TFTP then write to SD card.
      6. Load recovery system via TFTP then write to SD card.
      7. Install bootloader, recovery and production to NAND.
      8. Reboot.
      9. Reset all settings to factory defaults.
      0. Exit


  Press UP/DOWN to move, ENTER to select, ESC to quit
```

4. 注意力右上角方括号内容是`SD card`模式。如果你的终端能显示颜色，那么选项7应该是红色字体。按键盘上下键选择第七个选项，也就是安装Bootloader、Recovery和系统本身到NAND中。

5. 安装完成后按键盘RETURN键回到菜单。此时已经将OpenWrt系统安装到板载的SPI-NAND中了。

其实板子出厂自带的OpenWrt系统就在SPI-NAND中，但是可能无法开机进入系统，原因未知。至少我的OEM系统会启动失败。毕竟BPI公司的产品优势只有硬件看起来还不错，软件生态真是垃圾。一个产品的定位无论是面向开发者还是普通消费者，产品本身、产品生态和售后服务都应当做好，否则这个产品只能称之为半成品和工业垃圾。

6. 将开发板上的SW3开关切换到NAND模式（A0B1），之后按RST按钮重启到SPI-NAND的U-Boot选择菜单，并快速按上下键防止超时跳过。此时我们可以看到右上角方括号内显示的是`SPI-NAND`。按上下键选择选项9，安装Bootloader、Recovery和系统本身到eMMC中。

这个步骤不移除TF卡其实也没关系，不会影响SPI-NAND中的系统安装到板载eMMC中。可能是启动到板载SPI-NAND或eMMC中的系统时，电路层面屏蔽了TF卡的存在。因为当你插着TF卡启动到板载系统中并且用`lsblk`命令（系统默认没有，需要手动安装），你会发现识别到`mmcblk0`的大小就是板载eMMC的大小。这意味着你无法通过板载系统反向安装到TF卡内。

7. 安装完成后按键盘RETURN键回到菜单。此时已经将OpenWrt系统安装到板载的eMMC中。

8. 将板子上SW3开关切换到A1B0，也就是eMMC模式。按RST重启，直接进入OpenWrt系统就可以了。

## 使用M.2 NVME固态硬盘挂载 /overlay 分区和交换分区

BPI-R4只支持NVME协议的M.2接口的固态硬盘。你可以进入板载系统给M.2固态分区并挂载，但是你无法在板载系统中给TF卡分区。原因前面已经说了，这个章节就只简单讲下如何将`/overlay`分区挂载到M.2固态硬盘上。将其他块设备的分区挂载到`/overlay`分区下也是类似的操作。**给设备分区会删除所有数据，请操作前备份！**

1. 将BPI-R4的LAN口与电脑连接，电脑输入命令`ssh root@192.168.1.1`SSH连接进入OpenWrt的SHELL环境。按照之前的教程联网后，安装所需的内核模块和工具（你可以在网页中安装软件，但最后还是需要用命令分区，除非在插入NVME固态前已经弄好了）：

```bash
opkg update
opkg install kmod-nvme gdisk lsblk kmod-fs-f2fs block-mount
```

`gdisk`与传统的`fdisk`类似，只不过`fdisk`专门用于MBR分区。在现代操作系统中，MBR分区将被逐渐淘汰，取而代之的则是GPT分区。`gdisk`正是专门为GPT分区的设备打造，而且使用方式与`fdisk`类似。

我打算将分区格式化为F2FS系统所以需要安装`kmod-fs-f2fs`文件系统模块。如果你需要其他类型的文件系统，安装对应模块即可。

2. 输入命令`lsblk`查看当前的分区表，确认是否有一个以`nvme`开头的TYPE是disk的块设备。如果没有，就检查一下你的固态硬盘是否插好、是否损坏（也有可能不兼容）。我这里显示的设备名称是`nvme0n1`，那么接下来就用`gdisk`将这个硬盘分区。以下是操作记录：

```txt
root@OpenWrt:~# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
mtdblock0    31:0    0     2M  1 disk
mtdblock1    31:1    0   126M  0 disk
mmcblk0     179:0    0  59.5G  0 disk
├─mmcblk0p1 179:1    0     4M  0 part
├─mmcblk0p2 179:2    0   512K  0 part
├─mmcblk0p3 179:3    0     2M  0 part
├─mmcblk0p4 179:4    0     4M  0 part
├─mmcblk0p5 179:5    0    32M  0 part
├─mmcblk0p6 179:6    0    20M  0 part
└─mmcblk0p7 179:7    0   448M  0 part
ubiblock0_4 254:0    0  16.2M  0 disk
fit0        259:0    0  10.7M  1 disk /rom
fitrw       259:1    0 431.9M  0 disk /overlay
nvme0n1     259:2    0 119.2G  0 disk


root@OpenWrt:~# gdisk /dev/nvme0n1
GPT fdisk (gdisk) version 1.0.10

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help): ?
b	back up GPT data to a file
c	change a partition's name
d	delete a partition
i	show detailed information on a partition
l	list known partition types
n	add a new partition
o	create a new empty GUID partition table (GPT)
p	print the partition table
q	quit without saving changes
r	recovery and transformation options (experts only)
s	sort partitions
t	change a partition's type code
v	verify disk
w	write table to disk and exit
x	extra functionality (experts only)
?	print this menu

Command (? for help): o
This option deletes all partitions and creates a new protective MBR.
Proceed? (Y/N): y

Command (? for help): n
Partition number (1-128, default 1): 1
First sector (34-250069646, default = 2048) or {+-}size{KMGTP}:
Last sector (2048-250069646, default = 250068991) or {+-}size{KMGTP}: +2G
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300): L
Type search string, or <Enter> to show all codes: swap
8200 Linux swap                          a502 FreeBSD swap
a582 Midnight BSD swap                   a901 NetBSD swap
bf02 Solaris swap
Hex code or GUID (L to show codes, Enter = 8300): 8200
Changed type of partition to 'Linux swap'

Command (? for help): n
Partition number (2-128, default 2): 2
First sector (34-250069646, default = 4196352) or {+-}size{KMGTP}:
Last sector (4196352-250069646, default = 250068991) or {+-}size{KMGTP}:
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300):
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/nvme0n1: 250069680 sectors, 119.2 GiB
Model: LITEON CA3-8D128-HP
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 806EE748-FC39-487B-BB07-FE0F2D03E47E
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 250069646
Partitions will be aligned on 2048-sector boundaries
Total free space is 2669 sectors (1.3 MiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         4196351   2.0 GiB     8200  Linux swap
   2         4196352       250068991   117.2 GiB   8300  Linux filesystem

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N): y
OK; writing new GUID partition table (GPT) to /dev/nvme0n1.
The operation has completed successfully.


root@OpenWrt:~# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
mtdblock0    31:0    0     2M  1 disk
mtdblock1    31:1    0   126M  0 disk
mmcblk0     179:0    0  59.5G  0 disk
├─mmcblk0p1 179:1    0     4M  0 part
├─mmcblk0p2 179:2    0   512K  0 part
├─mmcblk0p3 179:3    0     2M  0 part
├─mmcblk0p4 179:4    0     4M  0 part
├─mmcblk0p5 179:5    0    32M  0 part
├─mmcblk0p6 179:6    0    20M  0 part
└─mmcblk0p7 179:7    0   448M  0 part
ubiblock0_4 254:0    0  16.2M  0 disk
fit0        259:0    0  10.7M  1 disk /rom
fitrw       259:1    0 431.9M  0 disk /overlay
nvme0n1     259:2    0 119.2G  0 disk
├─nvme0n1p1 259:3    0     2G  0 part
└─nvme0n1p2 259:4    0 117.2G  0 part
```

这里使用了64GB TF卡中的系统来分区，所以`mmcblk0`的大小是59.5G。固态硬盘则是120G。我为固态硬盘重建了GPT分区表，并划分了2GB交换分区，其余空间作为挂载`/overlay`的数据分区。

3. 分区完成后，需要格式化一种文件系统才能使用。以下是操作记录：

```txt
root@OpenWrt:~# lsblk /dev/nvme0n1
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:2    0 119.2G  0 disk
├─nvme0n1p1 259:3    0     2G  0 part
└─nvme0n1p2 259:4    0 117.2G  0 part


root@OpenWrt:~# mkswap /dev/nvme0n1p1
Setting up swapspace version 1, size = 2147479552 bytes


root@OpenWrt:~# mkfs.f2fs /dev/nvme0n1p2

    F2FS-tools: mkfs.f2fs Ver: 1.16.0 (2023-04-11)

Info: Disable heap-based policy
Info: Debug level = 0
Info: Trim is enabled
Info: Segments per section = 1
Info: Sections per zone = 1
Info: sector size = 512
Info: total sectors = 245872640 (120055 MB)
Info: zone aligned segment0 blkaddr: 512
Info: format version with
  "Linux version 6.6.86 (builder@buildhost) (aarch64-openwrt-linux-musl-gcc (OpenWrt GCC 13.3.0 r28597-0425664679) 13.3.0, GNU ld (GNU Binutils) 2.42) #0 SMP Sun Apr 13 16:38:32 2025"
Info: [/dev/nvme0n1p2] Discarding device
Info: This device doesn't support BLKSECDISCARD
Info: Discarded 120055 MB
Info: Overprovision ratio = 0.420%
Info: Overprovision segments = 252 (GC reserved = 245)
Info: format successful


root@OpenWrt:~# lsblk -f /dev/nvme0n1
NAME        FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
nvme0n1
├─nvme0n1p1 swap
└─nvme0n1p2 f2fs               c52e4b65-4458-44c4-b8b6-81652385c921


root@OpenWrt:~# reboot
root@OpenWrt:~# Connection to 192.168.1.1 closed by remote host.
Connection to 192.168.1.1 closed.
```

4. 重启完成后，打开OpenWrt网站，进入 System -> Mount Points 设置挂载选项，如图所示：

![OpenWrt挂载overlay分区](images/OpenWrt挂载overlay分区.webp)

5. 然后按板子上的RST按钮再次重启。由于重启到新挂载的`/overlay`分区没有任何数据，所以需要重新执行第一个步骤重新配置网络和安装软件。这次可以直接在 System -> Software 中更新列表并安装软件。其中`kmod-nvme` `block-mount` `kmod-fs-f2fs`必须安装。完成后再次重启。

6. 重启完成后进入 System -> Mount Points 再次设置挂载选项。这次只需按照下图操作添加SWAP交换分区并保存：

![OpenWrt设置swap分区](images/OpenWrt设置swap分区.webp)

7. 再次重启系统确保完全生效。如图所示，SWAP 和 Storage 都成功生效：

![OpenWrt设置分区成功](images/OpenWrt设置分区成功.webp)

最后，如果你觉得板载8G eMMC中将近7G被浪费了，那么可以拿剩余的7G分成SWAP分区使用。这样的话NVME就全盘用来挂载`/overlay`也行。但是，板载eMMC与NVME相比，哪个I/O性能会更高呢？不管性能如何，板载存储本身就不适合反复I/O，毕竟无法直接插拔替换，损坏了会很麻烦。

## OpenWrt 固件选择器创建自定义固件

[OpenWrt Firmware Selector](https://firmware-selector.openwrt.org/?version=SNAPSHOT&target=mediatek%2Ffilogic&id=bananapi_bpi-r4) 除了可以下载默认构建的固件外，还可以自定义软件包并下载构建好的固件。以下是官方24.10.1版本默认安装的软件包：

```txt
base-files ca-bundle dnsmasq dropbear firewall4 fitblk fstools kmod-crypto-hw-safexcel kmod-gpio-button-hotplug kmod-leds-gpio kmod-nft-offload kmod-phy-aquantia libc libgcc libustream-mbedtls logd mtd netifd nftables odhcp6c odhcpd-ipv6only opkg ppp ppp-mod-pppoe procd-ujail uboot-envtools uci uclient-fetch urandom-seed urngd wpad-basic-mbedtls kmod-hwmon-pwmfan kmod-i2c-mux-pca954x kmod-eeprom-at24 kmod-mt7996-firmware kmod-mt7996-233-firmware kmod-rtc-pcf8563 kmod-sfp kmod-usb3 e2fsprogs f2fsck mkf2fs mt7988-wo-firmware luci
```

以下是我计划替换和增加的软件包：

```txt
dnsmasq-full    # 提供更全面的DNS和DHCP支持（替换dnsmasq/1.8MiB）
kmod-nvme       # NVME 固态驱动内核模块（180KiB）
kmod-fs-f2fs    # F2FS 文件系统内核模块（20KiB）
lsblk           # 提供 lsblk 命令（1.42MiB）
block-mount     # 提供 LuCI 挂载设备界面（150KiB）
gdisk           # 现代 GPT 分区表硬盘分区工具（2.21MiB）
pciutils        # 提供 lspci 命令（1.8MiB）
```

加上没有显示的依赖包，总大小估计8MB以内，没有超过SPI-NAND的剩余可用空间大小，够基本安装使用了。其他非必要软件包可以在OpenWrt安装完成之后再装，或者用Sysupgrade固件更新的方式安装包含大量软件包的固件。所以最终安装的软件包如下：

```txt
base-files ca-bundle dropbear firewall4 fitblk fstools kmod-crypto-hw-safexcel kmod-gpio-button-hotplug kmod-leds-gpio kmod-nft-offload kmod-phy-aquantia libc libgcc libustream-mbedtls logd mtd netifd nftables odhcp6c odhcpd-ipv6only opkg ppp ppp-mod-pppoe procd-ujail uboot-envtools uci uclient-fetch urandom-seed urngd wpad-basic-mbedtls kmod-hwmon-pwmfan kmod-i2c-mux-pca954x kmod-eeprom-at24 kmod-mt7996-firmware kmod-mt7996-233-firmware kmod-rtc-pcf8563 kmod-sfp kmod-usb3 e2fsprogs f2fsck mkf2fs mt7988-wo-firmware luci dnsmasq-full kmod-nvme kmod-fs-f2fs lsblk block-mount gdisk pciutils
```

除了软件包外，还有初始化脚本可以修改。点空白文本框右下角的齿轮会自动生成一份模板。由于我使用猫棒配置PPPoE联网，联网之后需要安装其他软件进一步配置，所以这个脚本没什么用，删除留空白就行。

最后点击 REQUEST BUILD 就会开始在服务器上构建，成功后在下面的Custom Downloads 中会列出所有可供下载的文件，选择需要的下载即可。

## 最后

与 BPI-R4 相关的 OpenWrt 心得基本上就这些，还有一些折腾系统本身的这里就不展开讲了，之后可能会专门写几篇讲系统本身的折腾心得。本文到此为止。

## 相关参考链接

- [某运营商光猫内置反诈插件](https://www.quji.org/archives/7223)
- [Banana Pi BPI-R4 Wiki](https://wiki.banana-pi.org/Banana_Pi_BPI-R4)
- [【BPI-R4】and SFP](https://forum.banana-pi.org/t/bpi-r4-and-sfp/16945/215)
- [The OpenWrt Flash Layout](https://openwrt.org/docs/techref/flash.layout)
- [【原】使用BPI-R4开发板实现5G上网、Wi-Fi AP、文件共享和Docker服务](https://www.cnblogs.com/think8848/p/17993286)
