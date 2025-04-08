---
title: 一加11刷入OxygenOS操作系统
date: 2025-4-8
tags:
  - 一加Oneplus
  - 刷机
  - Android
  - OxygenOS
  - 教程
---

## 刷机的缘由

我这台一加11使用了一年多。硬件方面我比较满意，就是软件广告太多，ColorOS本身还会收集大量个人数据，比如小布AI的个人信息与偏好设置里明目张胆地用“自动学习中”的字眼含蓄地表示会收集最敏感的信息。对我来说这是绝对不能允许的！虽然在手机第一次使用时会明确告知收集哪些数据并且提供了开关供用户关闭，可这不足以表示这些数据不会被以其他“特殊的”方式被利用。假设你是一位维权人士，在正当维权期间被相关部门、相关企业恶意举报，导致手机某些功能被禁用或被操控（可能是摄像头、麦克风、联网功能等）。这不是没有可能的！2024年就有爆料某知名车企把维权用户的电动汽车远程锁定！电动汽车企业能有如此卑劣的行径，那么我也以极度的恶意揣测知名手机厂商跟风效仿。

所以，刷机不仅仅是保护个人数据安全的方式，也是为了防止手机可能被远程控制和窃听。

相比于第三方ROM，目前对PHB110这个型号的手机刷机最方便的恐怕是OxygenOS了。因为有大量的参考资料可供查阅，包括刷机前备份、刷机步骤、救砖等。其次，官方ROM还可以回锁，即使不Root，日常使用也足够了。

## 刷机前的准备工作

1. 一加11（salami）手机一台，ColorOS 14，**已解锁、已备份重要分区和数据(教程看往期文章)**
2. 可以在Bootloader中被识别的数据线和电脑
3. 电脑装好 android-tools
4. **刷机前把重要数据备份**
5. 刷机教程：[全球首发：一加11 ColorOS及氧OS懒人包|TWRP|卡刷包](https://www.daxiaamu.com/8481/) 我的这篇文章只是按照教程实践了一遍，所以在刷机前先看原文。
6. c636f33129394276a19b0cd42a85f43972c868e8 TWRP-13-14-OP11-Color597-V1.5.img (sha1sum) 下载链接看上面的刷机教程
7. 22d2552ddc31458940aae2da8a46b35f135c448d Oneplus11_Super_OxygenOS 13.0.0 A.09 GLO.zip (sha1sum) 下载链接看上面的刷机教程
8. **先把文章看几遍再操作**

## 复制OxygenOS卡刷包到手机

先复制卡刷包到手机内部存储空间的根目录（/sdcard/）中。

## 刷入 TWRP

进入 Bootloader，让电脑的`fastboot devices`识别到手机，然后执行以下命令刷入 TWRP分区：

```bash
sudo fastboot flash recovery TWRP-13-14-OP11-Color597-V1.5.img
```

完成后，按音量键选择进入 Recovery。

## 刷入OxygenOS卡刷包

在TWRP中点击安装，选择之前复制到手机内部存储空间里的卡刷包。然后直接滑动滑块开始刷机，其他的选项不用管。

正常情况下，开始刷机后手机会突然黑屏重启到TWRP的运行OpenRecoveryScript脚本界面，然后继续刷机，直到多次重启并进入OxygenOS系统。

但是也有例外。我刷机时在第一次重启这个阶段遇到了TWRP没有继续之前的刷机步骤的情况。具体表现就是进入TWRP后显示主界面，而不是运行OpenRecoveryScript脚本界面。可能刷机过程被神秘力量中断了，这时候再尝试重新点击安装刷机包会发现内部存储空间空无一物。据此可以猜测刷机进行到这个阶段可能是重置手机数据，还没开始安装系统。尝试清除缓存后重启系统，可以进入ColorOS，但是连接WIFI这个步骤无法开启WIFI，说明之前刷机时重启还包括了刷入特定的分区。好在等待一会可以跳过WIFI连接，进入系统后再次复制刷机包进手机，然后再进入TWRP重新安装刷机包刷机，之后我就成功进入OxygenOS 系统了。

刷机完成后，TWRP会被官方Recovery覆盖，这个需要注意。

## 确认手机型号

在设置 -> 关于本机界面可以看到，型号从原先的中国版PHB110变成了全球版CPH2449。

## 回锁 Bootloader（可选）

**注意：回锁有变砖风险，请保证官方系统未被修改！回锁前请备份好数据！**

进入 Bootloader 界面，电脑识别到手机后，使用以下命令回锁：

```bash
sudo fastboot flashing lock
```

按音量键选择锁定即可，回锁后会立即清除数据并重启。

## 结语

按大侠阿木的教程所述，刷入他的卡刷包后就可随意刷其他基于OxygenOS固件的第三方ROM了。原本我打算接着刷入CrDroid和KernelSU，方便更好地控制手机权限。后来用了一下OxygenOS感觉还不错，广告和数据收集都很少，预装APP主要是Google全家桶，于是就此作罢。而且回锁后还能微信支付宝指纹支付，光是这一点看来就比其他第三方系统和Root好了不少。

如果之后打算刷第三方系统了，我再另写一篇教程吧。

另外，其实网上有很多关于如何把 PHB110 转成 CPH2449 的教程，基本是几年前的帖子，时间太久远就不适合现在的系统，所以选择最新的方案或许是最合适的，尤其是有经验的人提出的方案。

## 相关参考资料

- [Stock Firmware Collection for OnePlus 11](https://xdaforums.com/t/stock-firmware-collection-for-oneplus-11.4543685/)
- [Step by Step EDL DownloadTool to restore your device from @KR5542's guide](https://xdaforums.com/t/step-by-step-edl-downloadtool-to-restore-your-device-from-kr5542s-guide.4709555/)
- [Android设备备份字库](https://mrwei95.github.io/2024/08/16/Backup-Flash-Memory/)
- [Reverting back to ColorOS (PHB110)](https://xdaforums.com/t/reverting-back-to-coloros-phb110.4589027/)
- [converting color os (PHB110) to OOS14 (2449) base on my own experience](https://xdaforums.com/t/converting-color-os-phb110-to-oos14-2449-base-on-my-own-experience.4674069/)
- [Sharing successful experience in converting Chinese OnePlus 11 to other versions](https://xdaforums.com/t/sharing-successful-experience-in-converting-chinese-oneplus-11-to-other-versions.4720211/)
- [Maybe a proper way to fix issues after converting Color to Oxygen](https://xdaforums.com/t/maybe-a-proper-way-to-fix-issues-after-converting-color-to-oxygen.4583321/)
- [Instructions for free unbricking of the device (suitable for A15)](https://xdaforums.com/t/instructions-for-free-unbricking-of-the-device-suitable-for-a15.4715178/)
- [A tricky way to use eSIM on CN/IN variant](https://xdaforums.com/t/a-tricky-way-to-use-esim-on-cn-in-variant.4609543/)
- [Converting PHB110 to CPH2449 and Install CRDROID 10 (Android 14)](https://xdaforums.com/t/converting-phb110-to-cph2449-and-install-crdroid-10-android-14.4710853/)
- [一加刷机衰退史——记录一下，我们是如何“温水煮青蛙”式的逐渐失去刷机自由的](https://www.daxiaamu.com/7625/)
- [【OP11】EDL DownloadTool to restore your device to OxygenOS/ColorOS](https://xdaforums.com/t/op11-edl-downloadtool-to-restore-your-device-to-oxygenos-coloros.4607995/)
- [Convert ColorOS to OxygenOS (OFP & OTA | Bootloader reLock & PIN working)](https://xdaforums.com/t/convert-coloros-to-oxygenos-ofp-ota-bootloader-relock-pin-working.4546967/)
- [Uotan Wiki · 刷机百科](https://wiki.uotan.cn/index.php?title=%E9%A6%96%E9%A1%B5)
- [大侠阿木 - 一加11 官方ROM下载](https://yun.daxiaamu.com/OnePlus_Roms/%E4%B8%80%E5%8A%A0OnePlus%2011/)
