---
title: 调整Debian/Ubuntu代码仓库优先级 - 一次Steam安装问题的解决过程
date: 2025-02-10 16:29:48
tags:
---

## 我的环境

Debian 12。根据nvidia官方文档，配置了nvidia的软件源，并使用apt的方式安装了Cuda和商业驱动

```shell
wget https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda-toolkit
sudo apt-get install -y cuda-drivers
```

这样做的优点是，将Cuda和驱动纳入了系统的源管理机制，方便后续升级 —— 但是，问题也出在这里……

## 出问题了

某年某月的某一天，我做了次大升级

```shell
sudo apt dist-upgrade
```

要下载的东西太多，之前配置的源太慢，我将它换成了清华的源（顺便说一句，清华确实牛逼！）

```
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm main contrib non-free-firmware
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bookworm-updates main contrib non-free-firmware
```

nvidia商业驱动也一起升级了。可是升级后，原本装好的Steam就打不开了。只能看见几个进程在跑，却没有窗口。

## 调查一下

查看steam log *~/.steam/debian-installation/logs/console-linux.txt*，发现问题了

```
X Error of failed request:  BadValue (integer parameter out of range for operation)
```

Bing告诉我，这是因为OpenGL相关的文件损坏了。考虑到之前升级了驱动，还切换过源镜像，猜测清华镜像里存在和新版本商业驱动不兼容的软件包。

又经过一番调查——其实就是不断重装——终于发现有一些软件包同时存在于清华镜像仓库和nvidia官方仓库中。具体的仓库信息可以通过`apt-cache policy`命令查看

```
# apt-cache policy libnvidia-egl-wayland1
 
libnvidia-egl-wayland1:
  已安装：1.1.13.1-1
  候选： 1.1.13.1-1
  版本列表：
 *** 1:1.1.10-1 500
        500 https://mirrors.tuna.tsinghua.edu.cn/debian bookworm/main amd64 Packages
     1.1.13.1-1 500
        500 https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64  Packages
        100 /var/lib/dpkg/status
```

在安装过程中，apt优先选择了清华仓库，最终造成了相关文件不兼容。

## 解决方法

只要保证所有显卡相关软件包都从nvidia仓库中下载，就可以解决上述问题。怎么做到呢？

可以配置不同软件仓库的优先级。首先查看有哪些仓库，以及他们的优先级。依然使用`apt-cache policy`命令
```
# apt-cache policy

软件包文件：
 100 /var/lib/dpkg/status
     release a=now
 500 https://developer.download.nvidia.com/compute/cuda/repos/debian12/x86_64  Packages
     release o=NVIDIA,l=NVIDIA CUDA,c=
     origin developer.download.nvidia.com
 500 https://mirrors.tuna.tsinghua.edu.cn/debian bookworm-updates/non-free-firmware amd64 Packages
     release v=12-updates,o=Debian,a=stable-updates,n=bookworm-updates,l=Debian,c=non-free-firmware,b=amd64
     origin mirrors.tuna.tsinghua.edu.cn
```

数字500就是仓库的优先级，origin可以认为是仓库的来源。

在/etc/apt/preferences.d下创建一个文件，我这里叫custom,内容为

```
Package: *
Pin: origin "developer.download.nvidia.com"
Pin-Priority: 700
```

意思是将origin为developer.download.nvidia.com的仓库优先级设为700。因为700>500，所以apt会先从nvidia仓库中下载软件包。

重装驱动，解决问题，steam又能正常地玩耍了。
