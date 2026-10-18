---
title: ThinkPad-Fedora笔记本常用安全加固
date: 2026-10-18
tags:
  - 年份-2026
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-Linux
  - 主题-ThinkPad
  - 主题-Fedora
  - 主题-BIOS
  - 主题-防火墙
  - 主题-指纹
  - 主题-LUKS2
  - 主题-加密
  - 主题-TPM2
  - 主题-4G模块
---

## 更新固件

用 `fwupdmgr` 可以方便地更新固件。执行以下命令获取更新。更新可能需要重复多次下面的命令；每次都要重启，尤其是从没更新过 `UEFI dbx` 的旧电脑。

```bash
sudo fwupdmgr update

```

如果你愿意，可以向 fwupd 组织匿名提交硬件信息~~（许愿适配）~~。

```bash
fwupdmgr report-devices

```

## BIOS 设置密码

BIOS 推荐设置两个密码：一个是【Power-On Password】，另一个是【Supervisor Password】。不建议这两种设置成一个密码，防止有人拿到了开机密码解锁 BIOS 设置并改密码。如果后续不想要密码了，在输入新密码时留空就能清除密码（大概，我没试过）。

- 【Power-On Password】顾名思义，主板上电就必须输入密码，防止别人拿到了笔记本换个硬盘就能用，或者用U盘引导加载live系统偷看数据。
- 【Supervisor Password】这个是BIOS密码，进入BIOS时需要输入的密码，防止别人拿到了笔记本修改BIOS的安全设置。

如果同时设置了【Power-On Password】和【Supervisor Password】，那么情况分以下两种：

1. 如果在主板上电的时候输入 **上电密码**，你可以直接进入系统，也可以进入BIOS界面，但无法修改安全相关的那部分设置。
2. 如果在主板上电的时候输入 **BIOS密码**，你可以直接进入系统（跟上电密码作用相同），也可以进入BIOS界面，但此时不必再输入一次BIOS密码，这就相当于同时输入了BIOS密码和主板上电密码，BIOS不会配置受限。

## 用 TPM2 自动解密被 LUKS2 加密的分区

自动解密的原理是LUKS2生成一个新密钥并将其存储在 TPM2 中，当笔记本状态发生变更，比如引导顺序变更、安全启动变更等、BIOS版本变更等，会使 TPM2 拒绝解密 LUKS2 分区。具体哪些变更会影响TPM2解密可以由PCR值控制，PCR值参考 `man systemd-cryptenroll`，通常建议使用`7,11,14`。

执行下面的命令启用 TPM2 自动解密。启用之后，请勿更新BIOS固件或修改BIOS安全设置等操作，可能导致TPM2内部存储的密钥失效。`by-uuid`后面接的一长串字符串就是希望被解密的LUKS2分区。

```bash
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7,11,14 /dev/disk/by-uuid/xxxxxxxxxx-xxxxxxxxxxxxxxxx-xxxxxxxxx

```

执行这个命令会提示你输入`sudo`密码，然后提示：

> 🔐 Please enter current passphrase for disk /dev/disk/by-uuid/xxxxxxxxxx-xxxxxxxxxxxxxxxx-xxxxxxxxx:

按照提示输入 LUKS2 密码，完成后提示：

> New TPM2 token enrolled as key slot 1.

说明成功创建新的LUKS2密钥并存储在 TPM2 的第一个槽位上。

## 指纹设置

Thinkpad 带指纹的笔记本可以用 `fprintd-enroll` 录入指纹，有一些不支持的指纹型号，如果遇到了，可能需要安装额外的Python包才能使用。

指纹模块通常走USB协议，执行 `lsusb` 即可查看指纹模块的ID和名称。幸运的是，我这款笔记本的指纹模块【Upek Biometric Touchchip/Touchstrip Fingerprint Sensor】`147e:2016` 早在多年前就被Linux社区支持了。

命令行执行 `fprintd-enroll`，可能弹框提示输入用户密码，然后按提示录入指纹。录入成功后，命令行执行 `fprintd-list $USER` 会列出当前用户所有已录入的指纹。指纹可以用于登录、解锁当前用户账户、使用`sudo`命令也会调用指纹模块进行识别：

> Swipe your right index finger across the fingerprint reader

此时按下指纹即可，跟手机指纹解锁操作一样。用指纹解锁能避免在他人面前输入`sudo`密码导致泄露的风险。

但是这在GNOME上有一个坑：如果重启系统或注销当前用户后，重新登录时使用指纹而非密码登录，大部分APP（如浏览器）会不断请求你输入登录密码。据说是没解锁 Keyring 导致APP无法获取当前用户的登录凭据。GNOME Keyring 的自动解锁依赖用户登录密码输入，指纹解锁没办法自动解开。

最近看到一个消息，Fedora45 开始用 oo7 逐步取代 GNOME Keyring，希望这个问题能以此解决吧。

## 4G模块防火墙配置

如果你加装了 4G LTE 上网模块，比如我用的是 【Quectel Wireless Solutions Co., Ltd. Quectel EM05-CE】，防火墙通常会使用默认设置。执行 `firewall-cmd --list-all` 列出默认的 FedoraWorkstation 防火墙，可以看到 `wwp0sxxxxx` 这个4G模块也用了 FedoraWorkstation。

问题是有些地区SIM卡获取到的4G网络可能是 IPV4公网和IPV6公网，这就导致如果你在系统中部署测试一些内网服务的端口，无意间暴露在了公网之上。IPV6还好说，如果真有IPV4公网，一天内端口爆破的机器人可数不胜数，就算没有安全隐患的端口，这流量也是分分钟给你耗光。

```bash
# 列出所有连接，找到 TYPE 是 gsm 的连接，比如我这里的名称是 `gsm-conn`
nmcli connection

# 列出 gsm-conn 这个连接使用的防火墙域
nmcli connection show gsm-conn | grep connection.zone

# 修改 gsm-conn 使用的防火墙为 `public`
nmcli connection modify gsm-conn connection.zone public

# 防火墙列出 `public` 域的配置信息
firewall-cmd --zone=public --list-all

# 防火墙配置 `public` 域移除 ssh 服务的入站流量
sudo firewall-cmd --permanent --zone=public --remove-service=ssh

# 重启防火墙以加载配置
firewall-cmd --reload

```
