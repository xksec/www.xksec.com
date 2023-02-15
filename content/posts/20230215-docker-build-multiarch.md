---
title: "在X86设备上构建多架构的容器"
date: 2023-02-15T17:03:51+08:00
draft: true
tags:
  - Linux
  - Docker
---

<!--more-->

升级内核





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

