---
title: YOLOv8训练模型转换成Sophgo（算能）BM1684X芯片可用的BModel格式
date: 2026-07-22
tags:
  - 年份-2026
  - 阶段-自由
  - 文体-配置教程
  - 篇幅-中长篇
  - 主题-Linux
  - 主题-Fedora
  - 主题-Podman
  - 主题-YOLO
  - 主题-Ultralytics
  - 主题-深度学习
  - 主题-BM1684X
  - 主题-算能
---

公司放在客户单位的YOLOv8训练服务器（Ubuntu 24.04）模型转换程序Core Dump了，U总折腾将近一个月，甚至让同事搞坏了公司里的训练服务器（Ubuntu 20.04）模型转换程序，仍旧没解决。我实在看不下去，忍痛腾出两天工作日的打游戏时间，主动免费加班，用公司留在宿舍的闲置ThinkPad笔记本装Fedora44 Sliverblue，把这问题解决了。并非炫耀自己有多厉害，只想表达这种问题连从没接触过深度学习训练的我都能解决，天天在我面前吹屄的U总水平也不过如此嘛，哈哈。

这篇文章属于半教程半踩坑记录，废话不多说，开始吧！

## 下载并导入容器

首先下载 `tpuc_dev:v3.4` 容器【[下载链接](https://sophon-assets.sophon.cn/sophon-prod-s3/drive/25/04/15/16/tpuc_dev_v3.4.tar.gz)】。[Sophgo（算能）官方文档](https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/tpu-mlir/quick_start/html/02_env.html#env-setup)给的链接是`v3.2`版本，比较旧，这里用[知乎这篇文章提供的链接](https://zhuanlan.zhihu.com/p/2040193786362450818)。计算哈希值，以下是我下载的文件的哈希值。

```bash
$ sha1sum tpuc_dev_v3.4.tar.gz | tee tpuc_dev_v3.4.tar.gz.sha1
5f367415e165c6098fd9e367910a8a403461950d  tpuc_dev_v3.4.tar.gz

```

解压并用 Podman 导入容器

```bash
gzip -d ./tpuc_dev_v3.4.tar.gz && podman load -i ./tpuc_dev_v3.4.tar

```

## 运行容器并配置Python环境

执行以下命令创建 `tpu-mlir-v1.28.1` 容器。注意 Fedora 的 SELinux。模型转换GPU环境不是必须，CPU就能转换。

```bash
mkdir ./workspace

podman run \
  --name tpu-mlir-v1.28.1 \
  -v $PWD/workspace:/workspace:z \
  -it sophgo/tpuc_dev:v3.4 /bin/bash

```

我试过了，直接启动容器来转换模型，YOLOv5 能转`.pt` 文件，但是 YOLOv8 转`.pt`和`.onnx` 都会报错 `Disabling PyTorch because PyTorch >= 2.4 is required but found 2.1.0+cpu`，所以我们要多做一些操作，把容器内的旧 PyTorch卸载，安装最新版本。

```bash
pip uninstall torch torchvision torchaudio

# 当前最新版本
# torch==2.13.0
# torchvision==0.28.0
# torchaudio==2.11.0
# 网络通畅的情况下用这个命令安装
# pip install torch torchvision torchaudio -i https://download.pytorch.org/whl/cpu
# 网络条件不好就用这个命令安装，但是全量下载，含CPU和GPU相关包。
pip install torch torchvision torchaudio -i https://pypi.tuna.tsinghua.edu.cn/simple

```

执行下面的命令下载 `tpu-mlir`，不用指定版本号也行。旧版本我试过行不通（在没更新PyTorch的情况下），不如用最新版本。

```bash
pip install 'tpu_mlir==1.28.1' -i https://pypi.tuna.tsinghua.edu.cn/simple

```

## 测试效果

从[Ultralytics Github](https://github.com/ultralytics/assets/releases#release-v8.4.0)下载v8.4.0版本的YOLOv8模型，`.pt`和`.onnx`版本都下载下来，并校验哈希值。我下载的是`yolov8s.onnx`。因为公司项目用的就是这个（f32，bm1684x）。然后将文件通过 `podman cp yolov8s.onnx tpu-mlir-v1.28.1:/workspace/` 命令复制进去，不要直接在主机里移动文件到绑定的目录里，SELinux 会阻止容器访问。

> 你可以下载`yolov8s.pt`试试转成`.bmodel`文件，我试过了，会报错：
> 
> ```txt
> RuntimeError: PytorchStreamReader failed locating file constants.pkl: file not found. This is an internal miniz error. If you are seeing this error, there is a high likelihood that your checkpoint file is corrupted. This can happen if the checkpoint was not saved properly, was transferred incorrectly, or the file was modified after saving.
> ```

用`.onnx`就没问题。执行以下代码转换模型，忽略生成的其他文件，只看有没有`xxx.bmodel`文件。

```bash
model_transform \
  --model_name yolov8s \
  --model_def ./yolov8s.onnx \
  --mlir yolov8s.mlir

model_deploy \
  --mlir yolov8s.mlir \
  --quantize F32 \
  --processor bm1684x \
  --model yolov8s_1684x_f32.bmodel
```

如果生成了 `yolov8s_1684x_f32.bmodel` 文件，就可以用公司YOLOv8训练服务器炼化的`.pt`文件转成`.onnx`，再用上述方法转成`.bmodel`，放进项目机子里跑了。

[算能官方文档](https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/tpu-mlir/quick_start/html/03_onnx.html)写的命令加了很多选项，还要提前下载什么资源文件。这些都不重要，用来测试图片识别效果的，可以看文档里参数功能表中哪些是必选。我提供的命令只包含必选项，可以照抄改个路径直接拿来用。

## 相关参考链接

- [TPU-MLIR快速入门手册](https://doc.sophgo.com/sdk-docs/v23.09.01-lts/docs_latest_release/docs/tpu-mlir/quick_start/html/index.html)
- [TPU-MLIR开发参考手册](https://sophgo.github.io/devoloper_manual/index.html)
- [Ultralytics Github](https://github.com/ultralytics/ultralytics)
- [算能技术资料](https://developer.sophgo.com/site/index/material/all/all.html)
- [sophgo/tpuc_dev DockerHub](https://hub.docker.com/r/sophgo/tpuc_dev/tags)
- [Get Start PyTorch](pytorch.org/get-started/locally/)
