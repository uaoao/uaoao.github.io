---
title: 摸索dvd+rw-tools刻录DVD-R光盘
date: 2026-9-25
tags:
  - 年份-2026
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-DVD
  - 主题-光盘
  - 主题-Linux
---

工作要刻录光盘，之前装Linux服务器尝试刻录了几次，平时刻录光盘都是在Windows虚拟机上刻录，所以花了几个小时研究Linux上如何刻录DVD-R的光盘。我家里的个人数据也打算买档案级光盘加密刻录，冷备份收藏起来。

## 安装

安装 `dvd+rw-tools`。

```bash
# Fedora
sudo dnf install dvd+rw-tools

# Fedora SilverBlue
sudo rpm-ostree install dvd+rw-tools

# Debian/Ubuntu
sudo apt install dvd+rw-tools

```


## 读取光盘信息

使用 `dvd+rw-mediainfo /dev/sr0` 读取光盘状态信息。

```txt
INQUIRY:                [ACER    ][DVD-RW AXD001   ][LD11]
GET [CURRENT] CONFIGURATION:
 Mounted Media:         1Bh, DVD+R
 Media ID:              CMC MAG/M01
 Current Write Speed:   8.0x1385=11080KB/s
 Write Speed #0:        8.0x1385=11080KB/s
 Write Speed #1:        6.0x1385=8310KB/s
 Write Speed #2:        4.0x1385=5540KB/s
 Write Speed #3:        3.0x1385=4155KB/s
 Speed Descriptor#0:    00/2295103 R@8.0x1385=11080KB/s W@8.0x1385=11080KB/s
 Speed Descriptor#1:    00/2295103 R@8.0x1385=11080KB/s W@6.0x1385=8310KB/s
 Speed Descriptor#2:    00/2295103 R@8.0x1385=11080KB/s W@4.0x1385=5540KB/s
 Speed Descriptor#3:    00/2295103 R@8.0x1385=11080KB/s W@3.0x1385=4155KB/s
READ DVD STRUCTURE[#0h]:
 Media Book Type:       00h, DVD-ROM book [revision 0]
 Legacy lead-out at:    2295104*2KB=4700372992
READ DISC INFORMATION:
 Disc status:           blank
 Number of Sessions:    1
 State of Last Session: empty
 "Next" Track:          1
 Number of Tracks:      1
READ TRACK INFORMATION[#1]:
 Track State:           blank
 Track Start Address:   0*2KB
 Next Writable Address: 0*2KB
 Free Blocks:           2295104*2KB
 Track Size:            2295104*2KB
 ROM Compatibility LBA: 270336
READ CAPACITY:          0*2048=0

```

参数【INQUIRY】表示我使用的光驱型号，现在用的是【Acer DVD-RW AXD001】，公司买的，用起来还行，支持USB2.0和Type-C接口。个人自用某东网购的 【ASUS ZenDrive SDRW-08V1M-U】支持千年光盘刻录，运行平稳很舒服，刻录的时候不会强烈震动，价格三百块左右，只有可内部收纳的Type-C线。

不同的光盘生产厂商，【Media ID】彼此不同，比如上文用的 aigo 光盘是`CMC MAG/M01`，莱德 ARITA DVD+R 光盘是 `RITEK/F16`。都是普通的 4.7G DVD 光盘（一桶50片，五十块左右）。注意，不是档案级光盘不适合冷备份那些需要存储长达二十年的数据。档案级光盘通常比这种普通光盘贵两倍多的价格，莱德档案级光盘（ISO/IEC 16963认证）DVD-R 是 `RITEKF1`，一桶50片，两百五十多块钱，这种存储介质价格高昂。如果见到那些宣称档案级光盘却非常便宜（一桶50片不到一百块），通常不是真的档案级光盘。另外还有莱德的千年DVD光盘 M-DISC，需要光驱本身支持刻录，光盘本身价格也相当贵，十块钱左右一张。光盘具体类别可以看哔站的一个视频，感觉讲得很好：[【光盘刻录教程】数据光盘刻录理论入门: 选择介质, 设备和文件系统 (2026 年 8 月)](https://www.bilibili.com/video/BV1itur6qExK/)

对于千年光盘，我觉得可以考虑刻录**非常重要的数据**，毕竟价格摆在那。目前我的方案是刻录重要数据用两张盘，一张五十块每桶的普通盘，和一张两百五十块每桶的档案级盘，刻录相同的数据（和数据的哈希值）。日常读取数据用普通盘，普通盘读取文件有问题再用档案级光盘重新刻录普通盘。当然，档案级光盘可以换用千年光盘，更放心，成本也更高，尤其是刻错了简直血亏。

注意看【Number of Sessions】、【"Next" Track】、【Number of Tracks】，空光盘的数值是`1`。每次向光盘附加刻录文件，这三个数值会被加一，比如下文初始化光盘并刻录一个文本文件。

## 初始化光盘并刻录文件

```bash
growisofs -Z /dev/sr0 -R -J /path/to/file.txt

```

首次刻录使用 `-Z`，表示初始化光盘并开始第一个会话；`-R`表示启用 Rock Ridge 扩展，保留 Unix 权限、长文件名等；`-J`表示启用 Joliet 扩展，提高 Windows 兼容性；可选的`-V <LABEL_NAME>` 可以设定光盘的名称

刻录显示信息如下所示：

```txt
Executing 'mkisofs -R -J /path/to/file.txt | builtin_dd of=/dev/sr0 obs=32k seek=0'
I: -input-charset not specified, using utf-8 (detected in locale settings)
Total translation table size: 0
Total rockridge attributes bytes: 247
Total directory bytes: 0
Path table size(bytes): 10
Max brk space used 0
182 extents written (0 MB)
/dev/sr0: "Current Write Speed" is 3.1x1352KBps.
builtin_dd: 192*2KB out @ average 0.0x1352KBps
/dev/sr0: flushing cache
/dev/sr0: closing track
/dev/sr0: closing session
/dev/sr0: reloading tray

```

实际上 `growisofs` 这个命令底层调用的是 `mkisofs` 命令，直接把文件打包成镜像刻录到了光盘，省去用户先打包镜像再刻录。

第一次刻录文件后，光盘信息如下：

```txt
INQUIRY:                [ACER    ][DVD-RW AXD001   ][LD11]
GET [CURRENT] CONFIGURATION:
 Mounted Media:         1Bh, DVD+R
 Media ID:              CMC MAG/M01
 Current Write Speed:   8.0x1385=11080KB/s
 Write Speed #0:        8.0x1385=11080KB/s
 Write Speed #1:        6.0x1385=8310KB/s
 Write Speed #2:        4.0x1385=5540KB/s
 Write Speed #3:        3.0x1385=4155KB/s
 Speed Descriptor#0:    00/191 R@4.0x1385=5540KB/s W@8.0x1385=11080KB/s
 Speed Descriptor#1:    00/191 R@4.0x1385=5540KB/s W@6.0x1385=8310KB/s
 Speed Descriptor#2:    00/191 R@4.0x1385=5540KB/s W@4.0x1385=5540KB/s
 Speed Descriptor#3:    00/191 R@4.0x1385=5540KB/s W@3.0x1385=4155KB/s
READ DVD STRUCTURE[#0h]:
 Media Book Type:       00h, DVD-ROM book [revision 0]
 Legacy lead-out at:    2295104*2KB=4700372992
READ DISC INFORMATION:
 Disc status:           appendable
 Number of Sessions:    2
 State of Last Session: empty
 "Next" Track:          2
 Number of Tracks:      2
READ TRACK INFORMATION[#1]:
 Track State:           invisible
 Track Start Address:   0*2KB
 Free Blocks:           0*2KB
 Track Size:            192*2KB
 ROM Compatibility LBA: 270336
READ TRACK INFORMATION[#2]:
 Track State:           blank
 Track Start Address:   2240*2KB
 Next Writable Address: 2240*2KB
 Free Blocks:           2292864*2KB
 Track Size:            2292864*2KB
 ROM Compatibility LBA: 270336
FABRICATED TOC:
 Track#1  :             17@0
 Track#AA :             17@192
 Multi-session Info:    #1@0
READ CAPACITY:          192*2048=393216

```

刻录完成后光盘将被自动弹出，再次插入读取光盘信息，可以看到，第一个Track `READ TRACK INFORMATION[#1]`现在被写入了数据，大小是 `192*2KB`。刻录的文件会存储在光盘的**根目录**。

## 挂载光盘到指定目录

通常Linux桌面操作系统会自动挂载到用户特定的目录，手动挂载和卸载方式如下：

```bash
# 挂载光盘
sudo mount /dev/sr0 /mnt/

# 卸载光盘
sudo umount /mnt/

```

## 附加刻录

第二次及之后的刻录使用 `-M`，表示在已有会话基础上继续写入。

```bash
growisofs -M /dev/sr0 -R -J /path/to/file2.txt

```

> 【提示】
> 重要规则：**追加刻录时，`-R`、`-J` 等文件系统选项应与首次刻录保持一致**。

如果你刻录一个文件夹，文件夹结构如下：

```txt
somedir/
├── rootfile.txt
└── subdir/
    └── subfile.txt

```

执行下面的命令刻录文件夹到光盘：

```bash
growisofs -M /dev/sr0 -R -J /path/to/somedir/

```

再次插入光盘，你会发现，`somedir/`消失了！取而代之的是文件夹中的内容，光盘中的文件夹结构如下：

```txt
./
├── rootfile.txt
└── subdir/
    └── subfile.txt

```

也就是说，**刻录的过程中会将文件夹内部内容直接刻录到光盘的根目录**，这一点请务必注意！

## 覆盖测试

如果刻录一个同名文件呢？比如前文刻录过 `somedir/rootfile.txt` 和 `somedir/subdir/subfile.txt` 都是在光盘的根目录，我们现在修改文件内容后再刻录上面的两个同结构、同名，但是不同内容的文件，看看会发生什么。

```bash
growisofs -M /dev/sr0 -R -J /path/to/somedir/

```

刻录过程输出如下：

```txt
Executing 'mkisofs -C 24416,36880 -M /dev/fd/3 -R -J /path/to/somedir/ | builtin_dd of=/dev/sr0 obs=32k seek=2305'
I: -input-charset not specified, using utf-8 (detected in locale settings)
Rock Ridge signatures found
Total translation table size: 0
Total rockridge attributes bytes: 1169
Total directory bytes: 4096
Path table size(bytes): 42
Max brk space used 0
37067 extents written (72 MB)
/dev/sr0: "Current Write Speed" is 8.2x1352KBps.
builtin_dd: 192*2KB out @ average 0.0x1352KBps
/dev/sr0: flushing cache
/dev/sr0: closing track
/dev/sr0: closing session
/dev/sr0: reloading tray

```

刻录成功了！读取这两个文件内容可以发现，原本的内容被覆盖成刚刻录的同名文件的内容！

嘿嘿，或许我们可以利用这个特性，刻录一个空的文件夹来清空只读光盘！

```bash
growisofs -M /dev/sr0 -R -J /path/to/emptydir/

```

刻录成功，但是插入光盘读取内容后发现，这根本没变化嘛！！！！

查了一下资料，说是额外添加一个选项，将光盘的根目录映射到我们的空文件夹，如下所示：

```bash
growisofs -M /dev/sr0 -R -J -graft-points /=/path/to/emptydir/

```

选项 `-graft-points` 的作用就是将 `=` 右侧的本地目录名称映射到光盘对应的目录。在这里，我们将一个空的文件夹映射到了光盘的根目录。

试了一下，还是不行。看样子只能覆盖文件，无法覆盖目录了。相比Windows上的UDF文件系统，Linux在光盘文件系统的读写支持相对而言不够全面。

查了一下 [archwiki](https://wiki.archlinuxcn.org/wiki/%E5%85%89%E7%9B%98%E9%A9%B1%E5%8A%A8%E5%99%A8)

> ISO-9960 多会话意味着一个介质使用只读文件系统的同时，依然可以在第一个未被使用的块地址写，并且在这个块上会有新的 ISO 树。这个新树协同于新加的或覆写的文件的数据块。数据的块在旧的 ISO 树中的，无法再被写。
> 
> Linux 和其他操作系统将挂载在介质上的最新的会话上的文件夹树。最新的树也将正常的显示旧会话的文件。 

大意是：旧会话的文件与新会话的文件都会被操作系统读取并显示。也就是说文件可以被覆盖，文件夹不行？

再做个测试，我们之前测的覆盖根目录，既然光盘的根目录无法覆盖，我们来覆盖子目录试试？

```txt
erase_test/
├── rootfile.txt
└── subdir/
    ├── subfile.txt
    └── sub-subdir/

```

将上述结构刻录光盘：

```bash
growisofs -M /dev/sr0 -R -J /path/to/erase_test/

```

读取光盘，目录树是这样的：

```txt
./
├── rootfile.txt
└── subdir/
    ├── subfile.txt
    └── sub-subdir/
```

然后我们把 `erase_test/subdir/` 中的所有文件和目录删掉，再刻录光盘。读取光盘，目录树是这样的：

```txt
./
├── rootfile.txt
└── subdir/
    ├── subfile.txt
    └── sub-subdir/
```

好吧，看样子所有的目录都无法被覆盖，只能覆盖文件内容。

## 封盘

封盘（Finalize）是在光盘末尾写入 Lead-out 区域，并更新光盘的 Disc Control Block（DCB），将光盘状态从 appendable（可追加）改为 unappendable（不可追加）。

封盘后，光盘变成纯只读介质，永久禁止再次刻录，防止数据篡改。

> 【注意】
> 注意：封盘是**不可逆操作**，一旦封盘，任何数据都无法再写入、修改或删除。

执行下列命令进行封盘处理：

```bash
growisofs -M /dev/sr0=/dev/zero

```

将 `/dev/zero` 作为输入源，`growisofs -M` 会将零数据写入光盘的剩余可用空间，直到光盘写满或到达物理边界。写满后，光驱固件会自动关闭光盘，完成封盘。

如果全新光盘只烧录一个文件就封盘，可以试试在命令中添加 `-dvd-compat` 选项。

封盘后的光盘状态如下，注意看【Disc status】：

```txt
INQUIRY:                [ACER    ][DVD-RW AXD001   ][LD11]
GET [CURRENT] CONFIGURATION:
 Mounted Media:         1Bh, DVD+R
 Media ID:              CMC MAG/M01
 Current Write Speed:   8.0x1385=11080KB/s
 Write Speed #0:        8.0x1385=11080KB/s
 Write Speed #1:        6.0x1385=8310KB/s
 Write Speed #2:        4.0x1385=5540KB/s
 Write Speed #3:        3.0x1385=4155KB/s
 Speed Descriptor#0:    00/2295103 R@8.0x1385=11080KB/s W@8.0x1385=11080KB/s
 Speed Descriptor#1:    00/2295103 R@8.0x1385=11080KB/s W@6.0x1385=8310KB/s
 Speed Descriptor#2:    00/2295103 R@8.0x1385=11080KB/s W@4.0x1385=5540KB/s
 Speed Descriptor#3:    00/2295103 R@8.0x1385=11080KB/s W@3.0x1385=4155KB/s
READ DVD STRUCTURE[#0h]:
 Media Book Type:       00h, DVD-ROM book [revision 0]
 Legacy lead-out at:    2295104*2KB=4700372992
READ DISC INFORMATION:
 Disc status:           complete
 Number of Sessions:    25
 State of Last Session: complete
 Number of Tracks:      25
READ TRACK INFORMATION[#1]:
 Track State:           invisible
 Track Start Address:   0*2KB
 Free Blocks:           0*2KB
 Track Size:            192*2KB
 ROM Compatibility LBA: 270336
...

```

## 总结

利用 `dvd+rw-tools` 创建的 ISO-9960 光盘文件系统支持多会话刻录和覆盖同名文件，但不支持覆盖目录，无法删除已有文件。

如果将光盘用于冷备份，购买档案级只读光盘，不要用普通光盘。备份前将数据加密并用`sha1sum` 计算所有文件的哈希值保存成文件一并刻录。刻录完成后再用 `sha1sum -c *.sha1` 校验所有光盘中的文件是否通过。

CD光盘容量太小；BD光盘容量大，BD刻录机却更贵。所以相比 CD 和 BD 光盘，档案级 DVD 容量合理价格相对便宜，适合存储文档、图片、录音这些小文件。

## 相关参考链接

- [man growisofs](https://man.archlinux.org/man/growisofs.1)
- [光盘驱动器 Archlinux Wiki](https://wiki.archlinuxcn.org/wiki/%E5%85%89%E7%9B%98%E9%A9%B1%E5%8A%A8%E5%99%A8)
