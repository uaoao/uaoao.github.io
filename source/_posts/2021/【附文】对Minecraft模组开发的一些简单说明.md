---
title: 【附文】对Minecraft模组开发的一些简单说明
date: 2021-5-22
tags:
  - 年份-2021
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

TODO

## Forge模组制作教程推荐（高版本）

对于模组制作等教程，可以在MCBBS的[ 编程开发 ](https://www.mcbbs.net/forum-development-1.html)板块找到。

- MCBBS楼主@[FledgeXu](https://www.mcbbs.net/home.php?mod=space&uid=3247777)的教程：
    >[Neutrino——面向初学者者的 1.15 Mod 开发教程](https://www.mcbbs.net/thread-1034476-1-1.html)
- B站UP主@[Felis__Catus_](https://space.bilibili.com/10409277)的系列**视频教程**：
    >[【1.16.X】Minecraft JavaEdition丨模组开发教程 EP01 创建一个物品](https://www.bilibili.com/video/BV1Uv411W7uJ)

## 安装两个 JDK 的原因

之所以安装两个版本的JDK，是因为【Language Support for Java™ by Red Hat】这个拓展更新到 **0.65.0版本** 的原因。Eclipse 平台决定将 JDK11 作为 9 月发布的最低要求，而 VSCode 是依赖 eclipsejdt.ls 服务器的，所以需要更新到 JDK11。（摘自博主@[ElasticForce](https://blog.csdn.net/u014792301)在[《VSCode中不再支持JDK8的解决方案》](https://blog.csdn.net/u014792301/article/details/107575799)中的一段话）

但是 Forge 制作和加载模组一般都是 **使用JDK1.8版本**，所以需要安装两个 JDK，且将 VSCode 运行环境改为 JDK1.8。如果你需要在 VSCode 中使用 JDK11 的环境，只需将代码中的

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

更改为

```json
  "java.configuration.runtimes": [
    {
      "name": "JavaSE-1.8",
      "path": "C:\\Program Files\\Java\\jdk1.8.0_291",
    },
    {
      "name": "JavaSE-11",
      "path": "C:\\Program Files\\Java\\jdk-11.0.11",
      "default": true
    },
  ],

```

就行了。注意这里是 VSCode，Windows 的 CMD 窗口仍然是 JDK1.8，改 CMD 中用的版本要将系统环境变量 `JAVA_HOME` 的值 `%JAVA_HOME1.8%` 修改为 `%JAVA_HOME11%` 。

另外，Minecraft 从快照 21w19a 开始，使用 Java16 运行游戏，目前暂不清楚 Forge 的模组开发是否要跟着改。（其实你可以在安装 JDK 时不用 JDK11 而改用 JDK16 甚至 JDK17，但考虑到兼容性，不推荐这么做）

![mcmod百科的解释](images/2021/mcmod百科的解释.png)

## OpenJDK 与 OracleJDK 等其它 JDK 的区别

Oracle JDK 是基于 OpenJDK 源代码构建的，因此 Oracle JDK 和 OpenJDK 之间没有重大的技术差异，所以两者用于模组开发是没有区别的。至于为什么用 OracleJDK 介绍，笔者只是觉得 Minecraft Launcher 使用的 java1.8 是 Oracle 公司的，跟着用准没问题（1.17 版本的用的是微软自家的开源 JDK16）。

## 推荐安装的 VSCode 扩展

- 【Chinese (Simplified) Language Pack for Visual Studio Code】提供中文界面支持
- 【indent-rainbow】使缩进更加明显（强迫症勿用）
- 【TOML Language Support】提供.toml文件的语法支持
- 【Material Icon Theme】不错的图标主题
- 【GitLens — Git supercharged】版本控制工具git的扩展，界面化操作git
- 【Project Manager】项目管理器，方便管理多个项目

## MDK 离线包下载

离线包的使用教程参见下载的网站。

1. MCBBS楼主@[FledgeXu](https://www.mcbbs.net/home.php?mod=space&uid=3247777)的地址：<https://github.com/FledgeXu/ForgeGradleOffline>
2. MCBBS楼主@[耗子](https://www.mcbbs.net/home.php?mod=space&uid=42509)的地址：[Minecraft模组开发离线包](https://www.mcbbs.net/thread-896542-1-1.html)

## VSCode 与 Java 专业 IDE 对比

Visual Studio Code 支持不少的语言，因其轻巧、启动速度快、性能要求少，所以优势主要体现在查看代码，制作网页、插件方面。对于大型项目而言，VSCode就显得力不从心了：编程效率不高，插件漏洞较多，功能无法满足工作需求等等。毕竟，VSCode的定位就是一款轻量级的编辑器，有再强大的扩展，也难以达到专业 IDE 的水平。

Eclipse、IntelliJ IDEA 等专业的 IDE，不仅对大型项目调试有极佳的优化，而且代码提示到位，内容全面丰富，工具齐全又方便。缺点也很明显，就是不支持多语言，启动速度慢，占用资源多，对性能要求高，不适合打开单个文件。

总之，不同的软件要配合使用，才能达到最好的的工作编程效率。虽然笔者的文章写的是用 VSCode 做模组开发，但还是推荐大家练习使用专业的 IDE，熟练掌握能大幅度提高开发效率。

## 常见问题

### MDK导入失败

文中提到过，MDK导入需要用很长的时间，失败了就重启项目再试一试。一般是网络问题，可能是防火墙阻止了，如果关闭防火墙也没用那就 ~~科 学 上 网~~ **用离线包** 吧。

### MDK 导入成功但是运行 genVSCodeRuns 失败

经过测试，含有中文等非法字符的**项目名**会造成 Gradle 构建失败（用户文件夹显示为中文，底层却是英文）

![文件夹名称包含中文字符](images/2020/文件夹名称包含中文字符.png)

![项目目录包含中文字符](images/2020/项目目录包含中文字符.png)

但是项目存放路径中含有中文名（“新 建 文 件 夹”）可以成功启动游戏（**注意这里说的是能启动游戏，但配置文件中是乱码的，仍然存在其它潜在的问题**）。

### 导入、构建成功，但是游戏启动失败

![网友问题请教](images/2020/网友问题请教.png)

这个问题一般是 `.gradle` 文件夹挪动了位置，或是使用离线包而导致资源加载错误，手动修改 `.vscode/launch.json` 中**所有有关位置的参数**，然后重启项目试试。仍然不行就用全新的 MDK 重新导入。

![VSCode配置位置参数](images/2020/VSCode配置位置参数.png)

### 中文路径的问题

中文路径问题在前文的问题二和问题三中就已经提到了，但是需要提醒读者：**不要随意移动项目**！**项目路径不要有中文，否则会出现乱码**！（图中是问题二路径包含中文“新 建 文 件 夹”导致的配置文件乱码）

![VSCode配置乱码](images/2020/VSCode配置乱码.png)

### 修改游戏启动内存

~~懒得再搞了直接截现成的，配置文件见问题四。~~

![修改游戏启动内存](images/2020/修改游戏启动内存.png)

### 其它问题

为了撰写这些文章，笔者对 VSCode 的配置都是在虚拟机中全新配置的，没有安装过其它的拓展，如果出现前文没有提及的失败或报错，有可能是拓展之间冲突了，**禁用或卸载**与模组制作不相干的拓展，问题或许就能迎刃而解。

同时欢迎各位开发者前来留言分享自己的经历，为国内模组发展提供丰富的参考。

