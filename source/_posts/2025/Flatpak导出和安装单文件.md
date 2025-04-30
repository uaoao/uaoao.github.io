---
title: Flatpak导出和安装单文件教程
date: 2025-04-29
tags:
  - Flatpak
  - Linux
  - 教程
---

## 说明

Flatpak 大部分情况下要联网才能下载安装软件。因为除了会安装软件本体外，还会安装依赖库。但有时候，我们需要安装一些离线才能安装成功的软件。比如 Flatpak 版本的 Chrome 浏览器，在其[构建文件](https://github.com/flathub/com.google.Chrome/blob/ce2f8b71ece1eb7431f70e72328f6e9ff33d366b/com.google.Chrome.yaml#L100)中写明了需要联网下载 Chrome 本体。对于一些无法下载 Chrome 本体的人来说就只能离线安装了。这类软件通常是闭源软件，在本地安装过程中必须联网下载软件本体，其 Flatpak 版的本质就是一个开源的安装脚本。

这里介绍一下按照[官方文档](https://docs.flatpak.org/en/latest/single-file-bundles.html)离线安装的方法。离线安装需要在另一台能够下载软件本体的电脑上导出 Flatpak 单文件包。比如上文的Flatpak Chrome浏览器，可以在另一台能访问Google下载网址的电脑上直接安装后导出单文件到这台电脑上安装（单文件包不包括依赖库，这台电脑必须能联网安装依赖库或已有所需依赖库）。

还有一种情况是软件本体即使联网也无法下载，比如[Clash for Windows](https://flathub.org/apps/io.github.Fndroid.clash_for_windows)软件本体两年前已删库，只能靠着某台已安装的电脑导出单文件来安装，或者用清单文件加上软件包本体[自己构建](https://docs.flatpak.org/en/latest/building-introduction.html)。

## 导出到单文件

导出的单文件包包含了安装脚本和已经从互联网下载的软件本体或额外资源。如果你使用较新版本的 Flatpak，导出的文件中会默认包含依赖库的元数据，安装单文件包时会解析并下载依赖库。

这里我以 Clash for Windows 为例子说明如何导出和安装单文件包。导出之前，先确保当前环境和依赖库分支都是最新版本（即使EOL）。然后执行以下命令导出到单文件：

```bash
flatpak build-bundle /var/lib/flatpak/repo ~/Downloads/io.github.Fndroid.clash_for_windows.flatpak io.github.Fndroid.clash_for_windows stable
```

如果当前电脑使用的是较旧的Flatpak版本，可能需要添加`--runtime-repo`选项，使用以下命令导出：

```bash
flatpak build-bundle /var/lib/flatpak/repo ~/Downloads/io.github.Fndroid.clash_for_windows.flatpak io.github.Fndroid.clash_for_windows stable --runtime-repo=https://flathub.org/repo/flathub.flatpakrepo
```

然后计算哈希值：

```bash
sha1sum ~/Downloads/io.github.Fndroid.clash_for_windows.flatpak.sha1 > ~/Downloads/io.github.Fndroid.clash_for_windows.flatpak.sha1
```

如果你遇到以下报错，说明使用的分支不正确。Flatpak默认使用 `master` 分支导出，但我们安装的软件一般是使用 `stable` 分支：

```txt
error: Refspec 'app/io.github.Fndroid.clash_for_windows/x86_64/master' not found
```

## 从单文件安装

使用以下命令安装单文件包到未安装的电脑上：

```bash
flatpak install ~/Downloads/io.github.Fndroid.clash_for_windows.flatpak
```

Flatpak官方文档写了也可以用下面的命令安装，但是我尝试过后会报错最后无法安装，所以不要使用这条命令：

```bash
sudo flatpak build-import-bundle /var/lib/flatpak/repo ~/Downloads/io.github.Fndroid.clash_for_windows.flatpak
```

以下是报错信息：

```txt
(flatpak build-import-bundle:8013): GLib-CRITICAL **: 20:59:10.075: g_str_has_prefix: assertion 'str != NULL' failed

(flatpak build-import-bundle:8013): OSTree-CRITICAL **: 20:59:10.075: _ostree_repo_get_remote: assertion 'name != NULL' failed

(flatpak build-import-bundle:8013): GLib-CRITICAL **: 20:59:10.075: g_propagate_error: assertion 'src != NULL' failed
```

这个安装命令可能仅供开发调试使用，因为需要指定安装位置。


## 结语

除了 Clash for Windows，还有一些被EOL的软件，比如`org.ryujinx.Ryujinx`和`org.yuzu_emu.yuzu`虽然现在还能用命令行下载，但不保证永远可下载，Flathub已经搜索不到这两个软件了，赶紧导出单文件备份一波吧。


## 相关参考链接

- [Cannot build and export to a repository (instead of installing it directly with --install option) #1](https://github.com/flathub/electron-sample-app/issues/1)

- [Welcome to Flatpak’s documentation!](https://docs.flatpak.org/en/latest/index.html)

