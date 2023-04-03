---
title: "在M1芯片上使用Qemu安装Ubuntu"
date: 2023-04-03T15:43:27+08:00
draft: false
---

这里介绍一下在 MacOS M1 Chip 的设备上，如何使用qemu 运行ubuntu；不论是基于学习还是测试的目的，这篇文章都会很有用。

<!--more-->

# 安装Qemu

第一步就是安装`qemu`， 使用 [`homebrew`](https://brew.sh/) 安装qemu是很简单的事情：

```bash
$ brew install qemu
```

安装完成后，检查下可执行文件是否已经正确安装：

```bash
$ whereis qemu-system-aarch64
qemu-system-aarch64: /opt/homebrew/bin//qemu-system-aarch64

$ qemu-system-aarch64 --version
QEMU emulator version 7.2.0
Copyright (c) 2003-2022 Fabrice Bellard and the QEMU Project developers
```

# 下载Ubuntu 的ISO文件

如果使用qemu安装ubuntu，需要下载ubuntu的iso文件，m1 chip的设备，需要下载的ARM版本的文件：[传送门](https://ubuntu.com/download/server/arm)


# 下载EDK2 UEFI image

下载一个预先定制的镜像文件，作为qemu启动的BIOS，用来支持m1 chip 的设备。

[点击下载](https://gist.github.com/theboreddev/5f79f86a0f163e4a1f9df919da5eea20#:~:text=QEMU_EFI%2Dcb438b9%2Dedk2%2Dstable202011%2Dwith%2Dextra%2Dresolutions.tar.gz)

解压到qemu的工作目录：

```bash
$ tar xvf QEMU_EFI-cb438b9-edk2-stable202011-with-extra-resolutions.tar.gz
x ./QEMU_EFI.fd
x ./QEMU_VARS.fd

```

# 安装 Ubuntu Server

## 准备 qcow2 文件

```bash
$ qemu-img create -f qcow2 ubuntu.qcow2 50G
```

文件是按需分配的，一开始只会占用很少的空间，随着使用会逐渐增大。

## 启动安装

```bash
qemu-system-aarch64 \
    -machine virt,highmem=off \
    -accel hvf \
    -cpu host \
    -smp 4 \
    -m 3G \
    -bios QEMU_EFI.fd \
    -device virtio-gpu-pci \
    -display default,show-cursor=on \
    -device qemu-xhci \
    -device usb-kbd \
    -device usb-tablet \
    -device intel-hda \
    -device hda-duplex \
    -hda ./ubuntu.qcow2 -cdrom ubuntu-22.04.2-live-server-arm64.iso
```

执行完此命令会弹出一个安装Ubuntu的桌面，常规安装即可。

## 安装完成后，调整启动脚本，取消掉cdrom

```bash
qemu-system-aarch64 \
    -machine virt,highmem=off \
    -accel hvf \
    -cpu host \
    -smp 4 \
    -m 3G \
    -bios QEMU_EFI.fd \
    -device virtio-gpu-pci \
    -display default,show-cursor=on \
    -device qemu-xhci \
    -device usb-kbd \
    -device usb-tablet \
    -device intel-hda \
    -device hda-duplex \
    -hda ./ubuntu.qcow2 
```

这里可以使用预设的账号密码登录，安装SSH-Server

## 再次调整启动脚本，关闭Display，启动Network Forward

```bash
nohup qemu-system-aarch64 \
    -machine virt,highmem=off \
    -accel hvf \
    -cpu host \
    -smp 4 \
    -m 3G \
    -bios QEMU_EFI.fd \
    -device virtio-gpu-pci \
    -display none \
    -device qemu-xhci \
    -device usb-kbd \
    -device usb-tablet \
    -device intel-hda \
    -device hda-duplex \
    -hda ./ubuntu.qcow2 \
    -net user,hostfwd=tcp::2222-:22 -net nic > /dev/null 2>&1 &
```

如此启动后，即可以无桌面使用SSH访问本机的2222端口。

# 安装完毕

