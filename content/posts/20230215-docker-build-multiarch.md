---
title: "在X86设备上构建多架构的容器"
date: 2023-02-15T17:03:51+08:00
draft: true
tags:
  - Linux
  - Docker
---

构建支持不通架构CPU的容器镜像，是个比较棘手的事情。docker官方提供了一个基于CLI的插件buildx提供构建的扩展能力，可以在x86或arm64的设备上构建支持多CPU架构的容器镜像。

<!--more-->

最近三周一直在做信创的适配工作，将手里基于ubuntu的实例镜像调整成同时支持x86及arm64版本的镜像，陆陆续续碰到了很多坑，以及各种文档中没有记录的地方。这里列出操作一份教程，方便日后的工作。


# 升级内核

`docker buildx`扩展是在 `docker v18.09`版本之后添加进去的，同时依赖kernel的`binfmt`，我手里的设备是CentOS8，其自带的kernel是支持`binfmt`功能的，如果是CentOS7的话，则需要升级kernel到较新的版本。

# 开启 Docker 实验特性

如果docker的版本低于 `v20.10`的话，需要更改配置文件打开实验特性


```bash
$ docker version
Client: Docker Engine - Community
 Version:           23.0.1
 API version:       1.41 (downgraded from 1.42)
 Go version:        go1.19.5
 Git commit:        a5ee5b1
 Built:             Thu Feb  9 19:49:07 2023
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          20.10.21
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.18.7
  Git commit:       3056208
  Built:            Tue Oct 25 18:00:24 2022
  OS/Arch:          linux/amd64
  Experimental:     true
 containerd:
  Version:          1.6.11
  GitCommit:        d986545181c905378b0f90faa9c5eae3cbfa3755
 runc:
  Version:          1.1.4
  GitCommit:        v1.1.4-0-g5fd4c4d
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
[root@k8s-master anhk]#
```
```bash
$ docker buildx inspect
Name:   default
Driver: docker

Nodes:
Name:      default
Endpoint:  default
Status:    running
Buildkit:  20.10.21
Platforms: linux/amd64, linux/386
```

```bash
 yum install docker-buildx-plugin  docker-ce-cli
```

```bash
$ cat /etc/docker/daemon.json
{
    ...
    "experimental": true,
    ...
}
```

```bash
$ docker run --rm --privileged multiarch/qemu-user-static --reset --credential yes  --persistent yes
```



```bash
$ docker buildx inspect
Name:   default
Driver: docker

Nodes:
Name:      default
Endpoint:  default
Status:    running
Buildkit:  20.10.21
Platforms: linux/amd64, linux/386, linux/arm64, linux/riscv64, linux/ppc64le, linux/s390x, linux/arm/v7, linux/arm/v6
```

