---
title: Fedora41配置Intel-DG1显卡
date: 2025-04-22
tags:
  - Fedroa
  - Linux
  - 驱动
  - 显卡
---

## 前言

自从配了新电脑后，显示器不够用的问题一直困扰着我。除了新买的显示器外，家里还有一台老古董VGA显示器和一台便携式HDMI显示器一直闲置着。因为主板只提供了一个HDMI和一个DP口，只能接一个HDMI的主显示器。而且如今显卡市场相当混乱，最新的9000系列AMD和5000系列NVIDIA显卡最便宜都他妈的2K起步，甚至两三年前发布的显卡现在比原价还贵。

前几天下定决心要买个显卡。考虑到我多年来一直主用Linux操作系统，NVIDIA在驱动方面一直是个问题，而AMD显卡又买不起近几年发布的，Intel吧感觉虽然主打中低端市场，但是兼容性比较差。

最后花了250块大洋在某鱼买了块二手的蓝戟 Intel DG1 显卡。虽说I卡兼容性差，但是奔着买亮机卡的目的去买显卡的话，Intel或许是个性价比比较高的显卡。直到收到货后无法点亮插在显卡上的显示器。。。

当然，在买显卡之前我一直用的是 AMD R7 8700G 自带的核显，显示器接主板上。

## Intel DG1 显卡存在的问题

按照卖家的说法，这个显卡需要比较新的主板，而且要在主板上开启 Resizable Bar 和 Above 4G 功能，同时系统硬盘的分区表必须是GPT。这些功能在我的主板上都默认开启，所以理应可以正常使用。

结果怎么，开机后插在I卡上的两个显示器都不亮，重置了BIOS再开机也不行，弄得我一度怀疑这卡是不是坏了（主显示器可以亮，因为核显能正常用）。在网上查了大量资料，最后得出一个假设：Linux 6.13 内核可能不支持这张卡。用`lspci -v`命令可以看到内核识别到了这张卡，但是没有驱动；用`dmesg`命令查不到与显卡有关的报错。网上也有人用这张卡装Linux系统，他们自己编译内核。

于是我用一个空硬盘装了Windows10。刚装系统时I卡不会点亮，装完系统后更新驱动，两个接在I卡上的显示器就正常点亮了。

后来查资料发现一篇发表在这个月2号的文章说Intel DG1 显卡 Linux 驱动模块终于移除了实验性标志。。。我靠，这GPU是2021年生产的，直到现在才移除实验性标志？？？真是服了牙膏厂。好在一些资料表明这张I卡的 `xe` 和 `i915` 内核模块只是在实验性状态，并非完全不支持，而`xe`模块早在Linux 6.8内核中就被引入。接下来就开始配置让这张Intel DG1显卡在 Fedora 41 SliverBlue 上亮起来吧。

一些参考资料我放在文章底部了，有需要可以看看。

## 修改 Fedora Linux 内核启动参数

1. 先获取到 Intel DG1 显卡的制造商ID：

```txt
[root@host:~]# lspci -nn | grep -i vga
06:00.0 VGA compatible controller [0300]: Intel Corporation DG1 [Iris Xe Graphics] [8086:4908] (rev 01)
14:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Phoenix1 [1002:15bf] (rev 06)
```

可以看到第一个VGA设备是I卡，最后面方括号内的数字就是【Vendor ID：Device ID】。我们需要的是Device ID，也就是`4908`。

2. 执行以下命令，添加内核启动参数：

```bash
# 查看当前内核启动参数
sudo rpm-ostree kargs

# 添加 xe 内核模块对蓝戟 Intel DG1 显卡驱动的实验性支持
# 注意末尾的 4908 改为你系统中显示的设备ID
sudo rpm-ostree kargs --append-if-missing=xe.force_probe=4908
# 或者，你可以使用 i915 内核模块
#sudo rpm-ostree kargs --append-if-missing=i915.force_probe=4908
# 如果你已经设置了 xe 模块，想要尝试 i915 模块
#sudo rpm-ostree kargs --delete-if-present=xe.force_probe=4908 &&
#sudo rpm-ostree kargs --append-if-missing=i915.force_probe=4908

# 再次查看
sudo rpm-ostree kargs
# 重启试试
reboot
```

我使用的操作系统是 Fedora 41 SliverBlue。Fedora SliverBlue从第41代开始，把内核启动参数与GRUB本身的设置分开了，内核启动参数现在只能由`rpm-ostree`管理，而GRUB本身的设置（比如启动后GRUB菜单延时）需要用GRUB本身的命令去配置。

如果你用的不是像 Fedora SliverBlue 这种不可变发行版，那么可以用以下命令修改启动参数：

```bash
# 查看启动参数设置
cat /etc/default/grub | grep GRUB_CMDLINE_LINUX_DEFAULT

# 编辑 /etc/default/grub
# 在 GRUB_CMDLINE_LINUX_DEFAULT 末尾添加以下参数
# 用空格与前面的参数分隔
# xe.force_probe=4908 或者 i915.force_probe=4908

# 保存后重新生成启动配置
grub-mkconfig -o /boot/grub2/grub.cfg
```

我没有验证过Fedora SliverBlue以外的发行版的配置是否成功，所以在执行前，最好在文章下面参考资料里多查一查。

另外如果你看了文章下面的参考资料，他们可能会这样设置：`i915.force_probe=!4908` 其实这个参数没有必要。如果你的I卡能开箱即用`i915` 或 `xe` 模块，就不太可能会看到这篇文章。

## 多显示器配置 GDM

三块显示屏，系统启动后登陆窗口不一定在期望的显示器上。可以这样配置（仅限GDM）：

1. 在设置中调整显示器位置和主显示器设置，调整好后会生成一份配置文件。由于在设置中配置的状态只有登陆账户后才能被读取，想要在系统启动到登陆界面时就按照理想的方式显示的话，就需要把登陆后的配置复制成全局配置。

2. 执行以下命令，生成全局配置文件：

```bash
sudo cp -v ~/.config/monitors.xml /var/lib/gdm/.config/
sudo chown gdm:gdm /var/lib/gdm/.config/monitors.xml
sudo ls -l /var/lib/gdm/.config/monitors.xml
```

3. 重启一下看看。

## 结语

`i915` 与 `xe` 模块的区别：`xe`是 Intel 近几年新开发的内核模块，主要是支持比较新的显卡或核显；而 `i915` 据说有20多年历史了，就我使用感受来说，这个模块的支持程度比 `xe` 好一点，如果你喜欢用 [Resources](https://flathub.org/apps/net.nokyan.Resources)、[Mission Center](https://flathub.org/apps/io.missioncenter.MissionCenter)、[CPU-X](https://flathub.org/apps/io.github.thetumultuousunicornofdarkness.cpu-x) 之类的系统资源监视器，`i915` 模块能提供更多I卡的状态信息。但是在PCIe连接速率上，用`lspci -vvv`可以发现，两者都是PCIe1.0 x1 （正常情况下Windows10中则是PCIe4.0 x8）。总之两者对I卡的支持都很差。

对于硬件解码之类是否支持，这个我还没研究，就我使用感觉而言，Linux系统插上这个卡后只是通过PCIe将AMD核显渲染的画面复制给I卡显示，并非两个显示器的成像都由I卡渲染，有点类似于显示接口扩展的感觉。。。

如果之后有研究，再补充。

## 相关参考资料

- [Intel graphics - Archlinux Wiki](https://wiki.archlinux.org/title/Intel_graphics#Testing_the_new_experimental_Xe_driver)
- [使用内核 xe 驱动使 Intel DG1 可以在 Jellyfin 下解码](https://icarusradio.github.io/guides/xe-dg1-jellyfin)
- [Intel Linux Driver Finally Dropping The Experimental Flag For Original DG1 Graphics](https://www.phoronix.com/news/Intel-DG1-Linux-Force-Probe)
- [Kernel parameters - Archlinux Wiki](https://wiki.archlinux.org/title/Kernel_parameters)
- [Login Screen on wrong monitor](https://discussion.fedoraproject.org/t/login-screen-on-wrong-monitor/70285)
- [/etc/default/grub is missing on Silverblue 41 fresh install](https://discussion.fedoraproject.org/t/etc-default-grub-is-missing-on-silverblue-41-fresh-install/135344)
- [Working with the GRUB 2 Boot Loader](https://docs.fedoraproject.org/en-US/fedora/f40/system-administrators-guide/kernel-module-driver-configuration/Working_with_the_GRUB_2_Boot_Loader/)
