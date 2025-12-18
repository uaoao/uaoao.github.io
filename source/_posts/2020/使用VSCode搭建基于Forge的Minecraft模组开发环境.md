---
title: 使用VSCode搭建基于Forge的Minecraft模组开发环境
date: 2020-11-19
tags:
  - 年份-2020
  - 阶段-专科
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-技术知识
  - 主题-VSCode
  - 主题-编程
  - 主题-Minecraft
  - 主题-Forge
---

最近更新：2021年6月12日

## 前言

看到网络上没有使用 Visual Studio Code 开发模组的教程，于是笔者记录下自己的经验**供大家参考**，如果有错误或更好的建议烦请在评论区留言，谢谢！

目前的 Forge 模组开发主要使用 Ecplise 或者 IntelliJ IDEA，这些IDE的界面和操作对于没什么 Java 项目开发经验的初学者来说有些复杂难懂。如果你不仅仅在使用 Java 这一种语言，或是你觉得使用专业的 IDE 来制作 Mod 有些大材小用。。。。
不管怎样，只要你是想**用VSCode开发**，那么，这篇教程一定对你有所帮助！

本教程主要讲解的是在 Windows10-x64 系统下（~~没用过Linux和Mac~~）使用 VSCode 搭建基于 Forge 的 Minecraft（1.14.4-1.16.5）模组开发环境，**并不教授模组的制作**，Forge 模组制作参见其它教程。

如果您不懂编程，那么这篇文章可能不适合您；如果您打算学习 Java 编程，那么这篇文章的前半部分可以教您快速搭建 Java 项目的开发环境。

在文章的最后，笔者附上了一些常见问题的回答，读者可以自行跳转阅读[《【附文】对Minecraft模组开发的一些简单说明》](/2021/5/22/【附文】对Minecraft模组开发的一些简单说明.html)。

那么，我们开始吧！

## 下载和安装JDK

在 Minecraft 版本 `1.17` 之前，VSCode 要搭建 Forge 的开发环境需要同时安装两个 JDK 的版本，分别是 **jdk-8u291** 和 **jdk-11.0.11**，至于为什么是两个，附文中会提及。

- `jdk-8u291` 下载地址：<https://www.oracle.com/cn/java/technologies/javase/javase-jdk8-downloads.html>

![JDK-8u291下载地址](images/2020/JDK-8u291下载地址.png)

- `jdk-11.0.11` 下载地址：<https://www.oracle.com/java/technologies/javase-jdk11-downloads.html>

![JDK-11.0.11下载地址](images/2020/JDK-11.0.11下载地址.png)

需要登陆 Oracle 账户才能下载的话就自己创建一个好了，以后或许用得上。在附文中，笔者会在附文中简单解释 OpenJDK 与 OracleJDK 等其它 JDK 的关系。

下载完后就是安装了。先安装哪个都无所谓，全都直接点击下一步就OK，**不建议更改安装位置，以防止环境变量等参数配置错误**。

## 配置环境变量

安装完两个 JDK 后，开始配置 JDK 的系统变量。

步骤：按【Win】键，直接搜索 `编辑系统环境变量` 并打开。

![编辑Windows10系统环境变量](images/2020/编辑Windows10系统环境变量.png)

在下面点击**环境变量**，然后在系统变量处点击**新建**，你需要新建以下三个变量：

![三个环境变量](images/2020/三个环境变量.png)

其中 `JAVA_HOME` 的 **值** 所指向的是你需要使用的 JDK 版本。目前我们只需要使用 `jdk-8u291`（即 `jdk1.8.0_291`），如果你需要使用 **jdk-11.0.11**，只需将 `JAVA_HOME` 中的 **值** 改为 `%JAVA_HOME11%` 就行了。

新建完以上变量后，在系统变量中选中 `Path`，点击【编辑】，然后新建、删除以下两个变量，**千万注意不要删错了！**

![新建、删除环境变量](images/2020/新建、删除环境变量.png)

配置完系统变量后，我们来验证一下 JDK 环境是否搭建成功。按【Win-R】键输入 `cmd`，回车打开命令窗口，分别输入以下两个命令回车，Java 版本号与我们安装的对应，则表明你的 JDK 环境搭建成功：

```sh
C:\Users\admin> java -version
java version "1.8.0_291"
Java(TM) SE Runtime Environment (build 1.8.0_291-b10)
Java HotSpot(TM) 64-Bit Server VM (build 25.291-b10, mixed mode)

C:\Users\admin> javac -version
javac 1.8.0_291

```

## 下载和安装Visual Studio Code

下载地址：<https://code.visualstudio.com/#alt-downloads>

选择【System Installer 64bit】版下载：

![VSCode下载](images/2020/VSCode下载.png)

下载完安装，注意以下**其它**中的选项都勾上：

![VSCode安装选项](images/2020/VSCode安装选项.png)

## 配置Visual Studio Code

以下两个为**必须安装**，推荐安装的拓展（包括中文支持）在附文中会提及。

- 【Java Extension Pack】提供 Java 语言支持
- 【Gradle Extension Pack】提供 Gradle 构建工具支持

扩展安装完后**重启一下计算机**。

接下来是配置 VSCode 的 JDK 版本。打开VSCode，按【Ctrl-Shift-P】，搜索命令 `>open settings` 并打开，如图所示：

![VSCode打开配置文件](images/2020/VSCode打开配置文件.png)

在两个大括号之间添加以下代码：

```json
  "java.configuration.runtimes": [
    {
      "name": "JavaSE-1.8",
      "path": "C:\\Program Files\\Java\\jdk1.8.0_291",
      "default": true
    },
    {
      "name": "JavaSE-11",
      "path": "C:\\Program Files\\Java\\jdk-11.0.11",
    },
  ],

```

注意路径中的 **JDK版本** 是否与你自己的一样。更改 VSCode 使用的 Java 版本见附文。

效果如下图所示（如果已经有代码在里面不要删掉, 在末尾加个逗号再添加）：

![VSCode配置项](images/2020/VSCode配置项.png)

保存修改后重新打开 VSCode。

## 测试 Java 项目

接下来我们来试试是否成功：

按【Ctrl-Shift-P】，输入 `>java`，选择【Create Java Project...】命令，选择【No build tools】，选择一个放项目的文件夹，输入项目名并回车，这样就新建了一个 Java 项目。

然后在资源管理器中打开 `src/App.java`，按【F5】调试，输出结果为：

```txt
Hello, World!

```

![VSCode输出HelloWorld](images/2020/VSCode输出HelloWorld.png)

如果右下角弹出错误的弹窗，**重启VSCode**试试，不然则是读者前文的某步操作出了问题，重新检查设置或是问问度娘或评论区留言吧。

现在，我们安装和配置完成了 VSCode，接下来就是最难成功的部分：下载和导入 Forge MDK。

## 下载和导入 MDK

MDK 可以在 Forge 官网下载和其它网友推荐的地址下载。
Forge 官方网址：<https://files.minecraftforge.net/net/minecraftforge/forge/>

笔者这里选的是 **1.16.5推荐版** 来介绍（1.14.4-1.16.5都行），右键复制链接，新建一个标签页，粘贴后砍掉**最前面**的 `https://adfoc.us/serve/sitelinks/?id=271228&url=` 这一截，然后回车下载：

![下载ForgeMDK](images/2020/下载ForgeMDK.png)

下载后解压到一个用于存放所有MDK项目的文件夹中，**项目路径和项目文件夹不能含有中文字符或空格等非法符号**！（相关问题请见附文）

**右键**这个解压缩的文件夹，选择**通过Code打开**（这种打开方式是将该文件夹作为一个项目打开），然后右下角弹窗选择【Yes】（**防火墙也需要选择允许**）:

![MDK初始化](images/2020/MDK初始化.png)

导入的过程只有漫长的等待。

由于网络原因，以上过程会经常失败，失败需要**重启VSCode重新导入**。也可以使用网友们分享的地址下载**离线包**再导入。离线包下载见附文！

离线包**初次**用VSCode打开也需要花一段时间来导入，但不会太久，失败了同样**重新打开导入**。大家时间充裕的话尽量自己尝试导入。因为离线包的作者使用的 IDE 不尽相同，导入的结果也就不一样，容易出现各种问题。

当右下角显示赞（不转圈圈）左下角只有一个警告（没有错误）时表明导入成功：（笔者使用校园网居然一次就导入成功了！！！！）

![MDK依赖项导入成功](images/2020/MDK依赖项导入成功.png)

导入成功后，点击最左边的【Gradle】图标，然后在【GRADLE TASKS】中选择【fg_runs】文件夹，点击文件【genVSCodeRuns】右侧三角形运行（没有显示这些文件说明读者前文的操作还没有成功），构建成功后在终端提示如图：

![Gradle任务构建成功](images/2020/Gradle任务构建成功.png)

回到资源管理器，打开 `src/main/java/com/example/examplemod/ExampleMod.java` 文件：

![ExampleMod.java文件](images/2020/ExampleMod.java文件.png)

这是Mod源码，**是MDK用来占位的**，没有什么作用，读者自己开发 Mod 把这个删了就行。样例 Mod 中的一些内容也值得看看，其中的注释或许对读者有所帮助！

## 启动 Minecraft

现在，按下【F5】进行调试。Minecraft，启动！

![Minecraft成功启动](images/2020/Minecraft成功启动.png)

初次启动会**多花一段时间**，下次就快了。

~~哦，听着这美妙的音乐，我疲惫的心灵都得到了升华。。。。~~ 

来看看Mod列表：

![ModList](images/2020/ModList.png)

吼吼，Mod成功运行！
至此，本教程结束，感谢大家的耐心阅读，谢谢！

## 附文

由于本文的篇幅过长，附文将在[《【附文】对Minecraft模组开发的一些简单说明》](/2021/5/22/【附文】对Minecraft模组开发的一些简单说明.html)中进行说明。

