---
title: BananaPi-R4路由器开发板使用心得
date: 2025-4-16
tags:
  - BananaPi-R4(BPI-R4)
  - OpenWrt
  - 开发板
  - 路由器
  - 心得
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

5. TF卡写入系统镜像文件后插在BPI-R4上，把板子侧边的切换开关打在A1B1的位置，然后插电开机。

| SW3  | A   | B   |
| ---- | --- | --- |
| NAND | 0   | 1   |
| eMMC | 1   | 0   |
| SD   | 1   | 1   |
| Halt | 0   | 0   |

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

未完待续

## 相关参考链接

- [某运营商光猫内置反诈插件](https://www.quji.org/archives/7223)
- [Banana Pi BPI-R4 Wiki](https://wiki.banana-pi.org/Banana_Pi_BPI-R4)
- [【BPI-R4】and SFP](https://forum.banana-pi.org/t/bpi-r4-and-sfp/16945/215)
