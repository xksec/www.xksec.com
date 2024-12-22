---
title: "从坑中爬起：ESXi 8.0直通NVIDIA显卡的血泪经验"
date: 2024-12-17T00:00:00+08:00
draft: false
tags:
  - EXSI
---

    在虚拟化技术迅猛发展的今天，利用虚拟机高效管理和分配硬件资源已成为技术人员日常工作的重要组成部分。然而，当涉及到需要高性能图形处理的任务，如深度学习、3D建模或游戏服务器时，如何在虚拟机中实现NVIDIA显卡的直通（GPU Passthrough）成为一大挑战。本文将详细分享我在使用ESXi 8.0进行NVIDIA显卡直通过程中的各种坑与经验，帮助大家少走弯路，顺利实现高效的硬件利用。

    要顺利实现NVIDIA显卡的直通，首先需要具备合适的硬件环境。我配置的主板是华硕 B760i，CPU是14600KF。
华硕 B760i 主板支持最新的IOMMU技术（Intel VT-d），这对于GPU直通至关重要。搭配强劲的i5处理器，不仅能够提供充足的计算性能，还支持多线程和高频率运行，为虚拟化环境提供了坚实的基础。
<!--more-->

> 备注： 如果主板上的网卡不是Intel家的，感觉要出门左拐了。

```
我在这里养了一只猫猫，路过的朋友可以摸摸它
　　　　　　 ＿＿
　　　　　／＞　　フ
　　　　　|  　_　 _ l
　 　　　／` ミ＿꒳ノ
　　 　 /　　　 　 |
　　　 /　 ヽ　　 ﾉ
　 　 │　　|　|　|
　／￣|　　 |　|　|
　| (￣ヽ＿_ヽ_)__)
　＼二つ
一个赞撸一次
别摸死了
```

# 使用Ventoy制作启动盘
    在开始安装ESXi 8.0之前，我们需要制作一个可启动的安装盘。Ventoy是一个开源工具，可以轻松将ISO文件写入U盘，并支持多ISO启动。Ventoy的优势在于支持多ISO启动，无需重复格式化U盘，且操作简单，只需一次安装后拖放ISO文件即可使用。

    以下是使用Ventoy制作启动盘的步骤：

> 1. 下载Ventoy并安装到U盘。
> 2. 将最新的ESXi 8.0镜像文件VMware-VMvisor-Installer-8.0U3b-24280767.x86_64.iso复制到U盘中。
> 3. 通过Ventoy的多ISO功能，可以同时存储其他工具的ISO，方便后续使用。

# 关闭安全启动

    为了避免ESXi在控制台中出现安全启动相关的警告，需要在BIOS中关闭 安全启动（Secure Boot）。具体步骤如下：

> 1. 进入BIOS：开机时按下 Del 或 F2 键进入BIOS设置界面。
> 2. 导航至安全选项：找到 Security 或 Boot 选项卡。
> 3. 关闭安全启动：将 Secure Boot 选项设置为 Disabled。
> 4. 保存并退出：按提示保存设置并重启系统。

    关闭安全启动后，ESXi在启动时将不会因签名校验失败而弹出警告，从而实现更顺畅的启动过程。

# 修改ESXi参数

由于ESXi 8.0没有对12代以后的CPU做默认的支持，所以在安装ESXi 8.0过程中会遇到紫屏的问题。为了解决这一问题，需要对ESXi的启动参数进行修改。

<b>1. 修改启动参数</b>

在ESXi启动界面，按 Shift + O 进入启动选项编辑模式，然后添加以下参数：`cpuUniformityHardCheckPanic=FALSE`

完整的启动参数示例如下：`kernelopt=… cpuUniformityHardCheckPanic=FALSE`

<b>2. 启动后通过SSH调整系统配置</b>

    启动进入ESXi系统后，通过SSH连接到主机，并执行以下命令以永久应用修改：
```
esxcli system settings kernel set -s cpuUniformityHardCheckPanic -v FALSE
```

此命令将启动参数持久化，确保每次启动ESXi时都应用该设置，防止紫屏问题再次出现。

<b>3. 配置NVIDIA显卡直通</b>

为了在虚拟机中成功直通NVIDIA显卡，还需要进行特定的配置。以下步骤将指导你完成这一过程：
{{< image src="/images/20241217-esxi/20241215-01.png" width="80%" max-width="600px" title="lspci -v | grep -i nvidia" >}}

{{< image src="/images/20241217-esxi/20241215-02.png" width="80%" max-width="600px" title="lspci -v | grep -i nvidia -A1" >}}

步骤：
编辑ESXi配置文件：

通过SSH连接到ESXi主机，执行以下命令将显卡设备标记为直通设备：
```
echo '/device/0000:01:00.0/owner = "passthru"' >> /etc/vmware/esx.conf
```
其中，`0000:01:00.0`为NVIDIA显卡的PCI地址，请根据实际硬件调整。

更新Passthru映射文件：

为了确保显卡的各项功能能够正确直通，需在passthru.map文件中添加相关配置。执行以下命令：
```
echo '10de 2803 bridge false' >> /etc/vmware/passthru.map
echo '10de 2803 link false' >> /etc/vmware/passthru.map
echo '10de 2803 d3d0 false' >> /etc/vmware/passthru.map
```
这里的`10de 2803`代表NVIDIA显卡的厂商ID和设备ID，请根据你的显卡型号进行相应调整。

重启ESXi服务：

为使上述配置生效，需要重启ESXi的管理代理服务。执行以下命令：
```
/etc/init.d/hostd restart
/etc/init.d/vpxa restart
```
或者，您也可以选择重启整个ESXi主机。


# 创建虚拟机及配置参数

{{< image src="/images/20241217-esxi/20241215-03.png" width="80%" max-width="600px" title="预留全部内存" >}}

在“虚拟机设置”中，点击“添加其他设备” > “PCI设备”，在列表中找到并勾选NVIDIA显卡（例如，`0000:01:00.0`）。


{{< image src="/images/20241217-esxi/20241215-05.png" width="80%" max-width="600px" title="添加PCI设备" >}}

取消UEFI安全启动

{{< image src="/images/20241217-esxi/20241215-06.png" width="80%" max-width="600px" title="lspci -v | grep -i nvidia -A1" >}}

为了优化GPU直通的性能和兼容性，需要在虚拟机的高级配置中添加以下参数：

```
# 该参数用于隐藏虚拟化环境，使得虚拟机能够更好地识别并利用NVIDIA显卡。
hypervisor.cpuid.v0 = FALSE
# 启用64位内存映射输入/输出（MMIO），提高显卡的内存访问效率。
pciPassthru.use64bitMMIO = TRUE
# 设置64位MMIO的大小为32GB，确保显卡有足够的内存资源进行高性能计算和渲染任务。
pciPassthru.64bitMMIOSizeGB = 32
```


确认所有设置无误后，点击“完成”保存虚拟机配置。


# 安装驱动

```
#禁用nouveau
touch /etc/modprobe.d/blacklist-nvidia-nouveau.conf
cat >> /etc/modprobe.d/blacklist-nvidia-nouveau.conf << EOF
blacklist nouveau
options nouveau modeset=0
EOF

touch /etc/modprobe.d/nvidia.conf
cat >> /etc/modprobe.d/nvidia.conf << EOF
options nvidia NVreg_OpenRmEnableUnsupportedGpus=1
EOF

sudo update-initramfs -u
sudo reboot

wget "https://us.download.nvidia.cn/XFree86/Linux-x86_64/535.129.03/NVIDIA-Linux-x86_64-550.135.run" -O NVIDIA-Linux-x86_64-550.135.run

sudo apt install build-essential
sudo apt install pkg-config libglvnd-dev
sudo chmod u+x NVIDIA-Linux-x86_64-550.135.run
# 如不带 -m=kernel-open 参数，kernel日志中会出现 RmInitAdapter failed! (0x26:0x56:1482) 报错。
./NVIDIA-Linux-x86_64-550.135.run -m=kernel-open

```


# 总结

在ESXi 8.0上实现NVIDIA显卡的直通并非一蹴而就，过程中可能会遇到各种各样的问题和挑战。但通过耐心的调试、深入的学习以及社区资源的借鉴，最终可以充分利用硬件资源，实现高效的虚拟化应用。希望我的血泪经验能够为大家提供一些有价值的参考，助力各位在虚拟化的道路上顺利前行！
