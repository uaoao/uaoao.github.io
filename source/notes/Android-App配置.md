---
title: Android-App配置
type: notes
cover: img/notes.webp
---

## LineageOS 常用设置项

- 修复网络图标感叹号：

```bash
adb shell "settings put global captive_portal_http_url http://connect.rom.miui.com/generate_204"
adb shell "settings put global captive_portal_https_url https://connect.rom.miui.com/generate_204"

```

- 修复系统时间不同步：

```bash
adb shell "settings put global ntp_server ntp1.aliyun.com"

```

### 官方构建 APP

- 【[魔曰](https://github.com/SheepChef/Abracadabra)】加密通话。
- 【[阅读](https://github.com/gedoor/legado)（已过时）】/【[Legado](https://github.com/HapeLee/legado-with-MD3)】 使用 Material Design 3 主题的开源阅读APP，可配合MultiTTS实现非常舒服的离线听书功能。
- 【[Apatch](https://github.com/bmax121/APatch)】**ROOT** Android操作系统ROOT提权工具。相比 KernelSU 支持的内核版本更广泛，具体内核要求看官方文档。
- 【[Obtainium](https://github.com/ImranR98/Obtainium)】一个用于订阅官方仓库构建Release的APP。
- 【[MMRL](https://github.com/MMRLApp/MMRL)】**ROOT** Magisk/KernelSU模块管理工具。
- 【[WebUIX](https://github.com/MMRLApp/WebUI-X-Portable)】**ROOT** MMRL模块WebUI工具。
- 【[App Manager](https://github.com/MuntashirAkon/AppManager)】**ROOT** 顾名思义，用于管理Android系统上的所有APP。
- 【[Anki-Droid](https://github.com/ankidroid/Anki-Android)】大名鼎鼎的卡片记忆APP。
- 【[Shelter](https://gitea.angry.im/PeterCxy/Shelter)（已过时）】/【[Haven](https://github.com/Kenneth-Cho-InfoSec/Haven)】使用 Material Design 3 主题的工作分区管理APP。
- 【[Forkgram](https://github.com/Forkgram/TelegramAndroid)】TG开源第三方手机客户端，移除了TG私有组件添加了一些新特性和自定义项。通常需要设置Proxy，APP会主动避开VPN设置。
- 【[Nextcloud](https://github.com/nextcloud/android)】私有云手机APP。
- 【[Termux](https://github.com/termux/termux-app)】**ROOT** Android手机终端。
- 【[LocalSend](https://github.com/localsend/localsend)】跨平台局域网文件传输APP。
- 【[Unciv](https://github.com/yairm210/Unciv)】类《文明5》开源游戏。
- 【[VLC](https://code.videolan.org/videolan/vlc-android)】世界上最著名的播放器

## 其他渠道

- 【MultiTTS】Text To Speech 引擎，来自 <t.me/MultiTTS_channel> 由于作者失联且项目未开源，最后一个版本信息如下。请勿随意下载其他来源的安装包（源下载地址也未必可信，但我个人一直在用感觉没问题）。

```txt
d3f7736b1bf162829677c0772a344696f8f5940e99f45879aa5e26070d0416af  MultiTTS_1.8.3(84)-arm64-v8a-release.apk
015340bd59ac7770019c64be444d58160d6d4274ed8e28002c04e80d83a817c4  MultiTTS_1.8.3(84)-armeabi-v7a-release.apk

```

## 其他

- [FDroid Repository List](https://github.com/userkilled/FDroid-List-Repository)
- [rime 輸入方案和配置列表](https://github.com/ayaka14732/awesome-rime)
