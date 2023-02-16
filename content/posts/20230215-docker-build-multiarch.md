---
title: "在X86设备上构建多架构的容器"
date: 2023-02-15T17:03:51+08:00
draft: false
tags:
  - Linux
  - Docker
---

构建支持不同CPU架构的容器镜像，是个比较棘手的事情。

docker官方提供了一个基于CLI的插件buildx提供构建的扩展能力，可以在x86或arm64的设备上构建支持多CPU架构的容器镜像。

<!--more-->

# 前言

最近三周一直在做信创的适配工作，将手里基于ubuntu的实例镜像调整成同时支持x86及arm64版本的镜像。开始是在自己的M1芯片的电脑上进行ARM版的镜像适配，当编译成功后，将X86版与ARM版合并成一个镜像的时候总是无法成功，陆陆续续碰到了很多坑。就动了使用buildx制作多架构统一镜像的年头，这里记录下碰到的问题，以及各种文档中没有描述的地方，以方便日后的工作。

这里使用的组件列表如下：

> - [buildx](https://docs.docker.com/engine/reference/commandline/buildx/)：docker-cli的扩展命令，支持docker的实验特性
> - [qemu-user-static](https://github.com/multiarch/qemu-user-static)：通过QEMU和binfmt_misc启用多架构容器的执行能力

# 升级内核

由于`docker buildx`扩展是在 `docker v18.09`版本之后添加进去的，同时依赖kernel的`binfmt`，我手里的设备是CentOS8，其自带的kernel是支持`binfmt`功能的，如果是CentOS7的话，则需要升级kernel到较新的版本。

以前在[此篇文章](/posts/20221230-quick-install-kubernetes/)中介绍过更新内核的命令。

# 开启 Docker 实验特性

如果docker的版本低于 `v20.10`的话，需要更改配置文件打开实验特性，编辑docker的配置文件：
```bash
$ cat /etc/docker/daemon.json
{
    ...
    "experimental": true,
    ...
}
```

检查实验特性是否已经打开：

```bash
$ docker version | grep Experimental
  Experimental:     true
```

# 安装 buildx 扩展

**安装**：由于本机的docker是使用docker-ce安装的，故安装docker-ce的扩展版本：

```bash
 yum install docker-buildx-plugin  docker-ce-cli
```

**检查**：检查当前buildx支持的CPU架构：

```bash
$ docker buildx inspect | grep Platforms
Platforms: linux/amd64, linux/386
```

**安装模拟器**：安装`qemu`的模拟器，以在X86架构上支持ARM64的编译环境
```bash
$ docker run --rm --privileged multiarch/qemu-user-static --reset --credential yes  --persistent yes
```

**再次检查**：检查当前buildx支持的CPU架构：
```bash
$ docker buildx inspect | grep Platforms
Platforms: linux/amd64, linux/386, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/arm/v7, linux/arm/v6
```

# 构建容器镜像

## Dockerfile

`buildx`支持的环境变量，可以有助于我们编写`Dockerfile`

- TARGETPLATFORM: 构建镜像的目标平台，例如：`linux/amd64`, `linux/arm/v7`, `windows/amd64`
- TARGETOS: 构建镜像的目标系统，例如：`linux`, `windows`
- TARGETARCH: 构架镜像的目标CPU架构，也是我们这次主要使用的环境变量，例如：`amd64`, `arm64`
- TARGETVARIANT: TARGETPLATFORM的变种，例如`v7`

`Dockerfile`示例：
```bash
FROM hub.mydemo.com/openeuler:22.03-${TARGETARCH}

# 需使用ARG定义，才能在Body中使用
ARG TARGETARCH
ARG GolangBin=https://go.dev/dl/go1.20.1.linux-${TARGETARCH}.tar.gz

COPY openEuler.${TARGETARCH}.repo /etc/yum.repos.d/openEuler.repo

RUN yum makecache && yum install -y wget tar && \
  wget -O- ${GolangBin}  | tar xz --strip-components 1 -C /usr

WORKDIR /root
```

## 执行命令
`docker buildx`命令的用法如下：

```bash
$ docker buildx build --platform linux/arm64/v8,linux/amd64 --push -f Dockerfile -t myuser/test .
# --platform: 需要构建的CPU架构，支持arm64、amd64等
# --push: 由于在构建多架构的镜像可能存在tag覆盖的情况，所以官方要求使用--push参数推送到镜像仓库中
```

## 检查

检查构建的镜像是否支持多CPU架构：
```bash
$ docker manifest inspect myuser/test | grep architecture
            "architecture": "arm64",
            "architecture": "amd64",
```

# 问题&记录

1. 在x86的设备上使用模拟器编译ARM64架构的程序，会很慢，尤其是`go build`，真的需要耐心。
2. `TARGET*`系列的环境变量，默认只在FROM行生效，如果要在BODY中使用的话，需要用ARG再次定义。
3. 使用`docker buildx`进行构建时，docker会使用`moby/buildkit:buildx-stable-1`作为构建镜像的实体。