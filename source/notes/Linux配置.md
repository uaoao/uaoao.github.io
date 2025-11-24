---
title: Linux配置
type: notes
cover: img/notes.webp
---

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
2. 创建 udev 规则文件 `/etc/udev/rules.d/61-thinkpad-keyboard.rules`，添加以下内容：

```txt
SUBSYSTEM=="input", DRIVERS=="lenovo", RUN+="/bin/sh -c 'FILE=$(find /sys/devices/ -name fn_lock 2>/dev/null); test -f $FILE && chmod 666 $FILE && ln -f -s $FILE /dev/fnlock-switch'"

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
3. 创建 udev 规则文件 `/etc/udev/rules.d/62-keyboard-switch.rules`，添加以下内容：（**注意部分内容修改成你系统中显示的结果**）

```txt
ACTION=="add", ENV{ID_MODEL}=="ThinkPad_Compact_USB_Keyboard_with_TrackPoint", RUN+="/usr/sbin/udevadm trigger --action=remove /dev/input/event3"

ACTION=="remove", ENV{ID_MODEL}=="ThinkPad_Compact_USB_Keyboard_with_TrackPoint", RUN+="/usr/sbin/udevadm trigger --action=add /dev/input/event3"

```

4. 执行 `sudo udevadm control --reload && sudo udevadm trigger` 加载配置文件。

- [参考链接](https://www.linuxquestions.org/questions/linux-desktop-74/udev-not-doing-remove-rules-841733/#post4146764)

## Fedora SliverBlue 系统自带 Firefox 添加 OpenH264 解码支持

正常情况下新安装的 SliverBlue 不包含 OpenH264 解码器，Firefox 在某些视频网站无法播放视频。执行以下命令，然后在 Firefox 插件页面启用 OpenH264。

```bash
sudo rpm-ostree override remove noopenh264 --install openh264 --install mozilla-openh264
reboot

```

