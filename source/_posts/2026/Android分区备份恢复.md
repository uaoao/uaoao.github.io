---
title: Android分区备份恢复
date: 2026-10-6
tags:
  - 年份-2026
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-Android
  - 主题-备份
---

> 内容参考自【[Android设备备份字库](https://mrwei95.github.io/2024/08/16/Backup-Flash-Memory/)】已证实有效备份，恢复功能暂未测试（毕竟手机没变砖）。

> 【警告】
> 本文操作仅适用并测试过高通机型的分区备份操作，联发科设备本人未测试。

早在去年就写了一篇文章【[一加11Root以及备份分区教程](/2025/4/7/一加11Root以及备份分区教程.html)】讲如何备份 Oneplus11，最近发现家里的几台旧手机有网友做了LineageOS非官方版本适配，想刷上去用上最新系统，又担心手机变砖，就把分区备份的操作复盘一下，顺便验证是否能在旧手机上顺利备份。备份需要 **ROOT**权限操作，至少预留 20G 可用存储空间。

## 高通机型常见的设备关键分区介绍（AI生成，待考证）

- `modemst1` `modemst2`: 存储 IMEI、射频校准数据
- `fsc`: 文件系统缓存、关联 IMEI 恢复
- `fsg`: 文件系统组，含基带配置
- `persist`: 指纹传感器校准、传感器数据、DRM密钥
- `modem`: 基带固件本身
- `abl` `xbl` `hyp` `tz`: 安全启动链相关
- `keymaster` `cmnlib*` TEE 安全环境密钥

设备唯一的数据（IMEI、基带等）分散在多个分区，**必须全部备份、同时恢复**，仅恢复部分分区会导致 IMEI 丢失或信号异常。

采用 UFS 闪存的分区路径在 `/dev/block/bootdevice/by-name/`，联发科设备可能在 `/dev/block/by-name`。

## 提权并查看存储空间

登录到手机Shell并提权。如果存储空间不足 20G，不要进行后续操作。

```bash
adb shell
su

ls /dev/block/bootdevice/by-name/ || exit 1
df -h /sdcard
id

```

## 读取设备分区信息

除了 `userdata` 和 `cache`分区，其他的都备份。请按顺序一步步执行命令。如果有报错，不要再进行下一步操作了。

```bash
mkdir /sdcard/000_Backup

ls -1 /dev/block/bootdevice/by-name | grep -ixvE "userdata|cache" | while IFS= read -r name; do echo "dd if=/dev/block/bootdevice/by-name/$name of=/sdcard/000_Backup/$name.img" >> /sdcard/000_Backup/001_Backup.sh; echo "fastboot flash $name $name.img" >> /sdcard/000_Backup/002_Restore.sh; done

```

## 执行备份脚本

不要把文件夹打压缩包，这会占用超过 20G 的空间。复制到电脑上再打包。

```bash
sh /sdcard/000_Backup/001_Backup.sh

cd /sdcard/000_Backup && sha1sum *.img > /sdcard/000_Backup/003_Checksum.sha1

exit

```

## 复制备份的内容到电脑

以下命令在电脑本机操作。

```bash
adb pull sdcard/000_Backup
cd ./000_Backup && sha1sum -c ./003_Checksum.sha1
cd ../ && tar -acf ./your_phone_rom_name.tar.gz ./000_Backup/

```

以下命令在手机Shell中操作，删除手机中已提取到电脑的文件夹。

```bash
adb shell
su

rm -r /sdcard/000_Backup
exit

```

## 用备份的分区恢复

> 【警告】
> 未验证可行性，操作前请三思。

看原文，用 `fastboot erase <partition_name>` 擦除分区，用 `fastboot flash <partition_name>` 刷入分区。

