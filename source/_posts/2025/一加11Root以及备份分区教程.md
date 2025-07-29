---
title: 一加11Root以及备份分区教程
date: 2025-4-7
tags:
  - 年份-2025
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-技术知识
  - 主题-Oneplus 一加
  - 主题-Android 安卓
  - 主题-Root
  - 主题-ColorOS
---

## 获取 Root 的目的

我最终的目的是将一加11刷机更换其他类原生系统。至于为什么要刷机，之后的刷机文章中再说，这篇文章主要讲解如何获取Root权限以及备份分区。在刷机前，必须将手机 eMMC 中的重要分区（俗称字库）备份。而备份分区需要解开 Bootloader 锁以及获取 Root 权限。

相比于刷机，获取 Root 权限的风险不会太大。不过为了以防万一，该做好的准备工作还是需要做好。

## Root 方案选择

获取Root的方案当下主要有三种：

1. Magisk
2. KernelSU (KernelSU-Next)
3. Apatch

我这里就选择 KernelSU 的方案。其他方案也行，看自己的需求。

## Root 前应当做好的准备工作

1. 一加11（salami）手机一台，ColorOS 14最新版本。如果你是 ColorOS 15，请网上查阅有关 Bootloader 的资料，有可能这篇文章不适合你。
2. 数据线若干：C-to-C、C-to-A、USB3、USB2 每种类型都准备一根
3. 带有 USB2、USB3、Type-C 口的Linux电脑多台（不满足这些接口需求的可以试试多接口类型的拓展坞；Windows也可以，大概）
4. ColorOS 14 最新版本的全量包 [PHB110_14.0.0.813(CN01) C.59](<https://yun.daxiaamu.com/OnePlus_Roms/%E4%B8%80%E5%8A%A0OnePlus%2011/ColorOS%20PHB110_14.0.0.813(CN01)%20C.59/>) SHA1SUM = cf76cf6ac3c8a87a0ad506f4ab025512227f8f38
5. 解包工具 [payload-dumper-go](https://github.com/ssut/payload-dumper-go)
6. 电脑装好 android-tools
7. [KernelSU 官方文档](https://kernelsu.org/zh_CN/guide/what-is-kernelsu.html) 至少需要看一遍。
8. 手机装好 [KernelSU App](https://github.com/tiann/KernelSU/releases)，确定内核版本为 5.15-android13-8

## 解开 Bootloader 锁

1. 在手机设置-关于本机-版本号 这里点击多次启用开发者选项。
2. 在系统与更新-开发者选项中，开启OEM解锁和USB调试。
3. 用准备好的各种数据线测试连接手机和电脑，并执行以下命令查找设备：

```bash
sudo adb devices
```

注意：这个步骤我发现用USB2的C-to-A线电脑无法识别，但是USB3可以（我的数据线都是UGREEN）

4. 执行以下命令，让手机进入fastboot：

```bash
sudo adb -d reboot bootloader
```

5. 执行以下命令，查找手机设备：

```bash
sudo fastboot devices
```

注意：这个步骤我用USB3的C-to-A线无法识别了，即使换成USB2也不行。

我花了一整天时间找遍了中英全网都找不到具体什么原因：有人说是驱动问题，我就换了 Windows10 电脑打上各种能用得上的驱动，还是无法识别；有人说是数据线的问题，建议最好用C-to-A的线刷机，可我用的就是C-to-A啊且USB2和USB3都测试过了。还有人说9008线刷无视BL锁。。。

眼看已经天黑了，绝望的我尝试用C-to-C数据线（可能是USB2，UGREEN充电器附带的65W线）连上Linux电脑，然后反复执行 fastboot 查找设备，没用。即将放弃之时，最后一次尝试在手机 fastboot 模式（即bootloader模式）选择重启到bootloader，电脑不断执行查找命令，结果，奇迹出现啦！！！电脑识别到了设备！！！😭

我从来没有想到过折腾 Oneplus 能麻烦到这种地步。以前用过的 Oneplus 不论 USB3 还是 USB2 都能识别，不管 C-to-A 还是 C-to-C。

再补充一下，后来我用另一台电脑的Type-C 口连接上述用的同一条C-to-C数据线，发现无法识别。换回上述同一台电脑的Type-C口又可以识别了，十分玄学。

你可以用`sudo dmesg -w`命令来观察内核是否识别到了手机的 Bootloader。进入Bootloader后插拔一下数据线，如果识别到了，dmesg 会输出以下类似的记录：

```bash
[ 1526.022646] usb 1-1: new high-speed USB device number 106 using xhci_hcd
[ 1526.151664] usb 1-1: New USB device found, idVendor=18d1, idProduct=d00d, bcdDevice= 1.00
[ 1526.151673] usb 1-1: New USB device strings: Mfr=1, Product=2, SerialNumber=3
[ 1526.151678] usb 1-1: Product: Android
[ 1526.151682] usb 1-1: Manufacturer: Google
[ 1526.151685] usb 1-1: SerialNumber: 你手机的序列号
```

注意Manufacturer是Google，不是Oneplus。如果没有识别到，则不会输出类似信息。

6. 继续解锁吧：

```bash
sudo fastboot flashing unlock
```

当出现白色的英文提示你解锁会发生什么什么时，用音量键选择“UNLOCK THE BOOTLOADER”。解锁后建议不要再上锁了，一加11修改boot分区后上锁会完全变砖，只能9008线刷。

解锁后系统会清除数据并重启，**不要设置PIN码**，然后再打开ADB调试。

## 从官方全量包中提取 init_boot.img 文件

参考payload-dumper-go 官方的文档，提取全量包中的payload.bin文件，找到 init_boot.img 后复制到手机里。

注意，全量包必须与你手机的ColorOS系统版本相同！

## 修补并刷入 init_boot.img 文件

手机开机后，再重新打开开发者选项和USB调试。

然后安装KernelSU APP，APP 会提示你修补一个init_boot文件，按照操作修补完成后会生成一个新的文件，把这个新的复制到电脑上，然后将手机重启到Bootloader，识别到手机后使用以下命令刷入修补过的镜像文件：

```bash
sudo fastboot flash init_boot kernelsu修补过的文件名.img
```

## 为 Shell 获取 Root 权限

刷入完成后重启系统，打开 KernelSU APP，在超级用户选项卡右上角点击显示系统应用。

搜索关键字`shell`，然后把 com.android.shell 这个包名的 APP 赋予超级用户权限。

数据线连接电脑和手机，使用命令 `sudo adb shell` 进入手机内部命令行，然后执行`su`，再执行`whoami`。如果结果是root，则说明成功。

## 备份重要的分区

按照 [Android设备备份字库](https://mrwei95.github.io/2024/08/16/Backup-Flash-Memory/)这篇文章的提示，备份字库即可。

这里简要摘抄核心命令：

1. 登陆shell并获取root权限

```bash
adb shell
su
```

2. 创建备份文件夹以及读取设备分区信息

```bash
mkdir /sdcard/000_Backup
ls -1 /dev/block/bootdevice/by-name | grep -ixvE "userdata|cache" | while IFS= read -r name; do echo "dd if=/dev/block/bootdevice/by-name/$name of=/sdcard/000_Backup/$name.img" >> /sdcard/000_Backup/001_Backup.sh; echo "fastboot flash $name $name.img" >> /sdcard/000_Backup/002_Restore.bat; done
```

3. 执行备份脚本

```bash
sh /sdcard/000_Backup/001_Backup.sh
```

4. 创建文件哈希校验表

```bash
cd /sdcard/000_Backup && sha1sum * > /sdcard/000_Backup/003_Sha1sum_list.txt
```

5. 修改bat脚本内容后打包分区

```bash
sed -i -e '/ super.img/s/^/::/g' -e '/ system.img/s/^/::/g' -e '/ system_a.img/s/^/::/g' -e '/ system_b.img/s/^/::/g' -e '/ vendor.img/s/^/::/g' -e '/ vendor_a.img/s/^/::/g' -e '/ vendor_b.img/s/^/::/g' -e '/ mmcblk0.img/s/^/::/g' -e '/ sda.img/s/^/::/g' -e '/ sdb.img/s/^/::/g' -e '/ sdc.img/s/^/::/g' -e '/ sdd.img/s/^/::/g' -e '/ sde.img/s/^/::/g' -e '/ sdf.img/s/^/::/g' -e '/ sdg.img/s/^/::/g' /sdcard/000_Backup/002_Restore.bat
cd /sdcard && tar -zcpvf PartitionBackup.tar.gz 000_Backup
sha1sum PartitionBackup.tar.gz > PartitionBackup.tar.gz.sha1
```

## 恢复某分区

如果之后意外损坏的某重要分区，可以通过`fastboot flash 分区名称 镜像文件`命令来恢复。

## 相关参考资料

- [一加11解锁后无法设置锁屏密码的处理方法：100%成功](https://www.daxiaamu.com/7601/)
- [ONEPLUS 11 Rooting Guide on Android 14](https://xdaforums.com/t/oneplus-11-rooting-guide-on-android-14.4632983/)
