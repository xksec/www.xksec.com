---
title: "监控Linux文件被进程篡改"
date: 2023-01-06T10:38:20+08:00
draft: false
tags:
  - Linux
---

最近有一台隔离环境的设备，sshd_config文件总是被修改，造成登录时间超过30秒。

这里介绍使用audit监控该文件被修改的事件。

<!--more-->

# 安装audit

```bash
# CentOS
$ yum install -y audit

# Ubuntu
$ apt install -y audit
```

# 创建监控规则

这里使用 `auditctl` 命令来添加监控规则：

```bash
# 查看规则列表
$ auditctl -l

# 添加监控规则
#   -w: 监控的文件
#   -p: 监控的事件, r=read, w=write, a=append, x=execute
#   -k: 监控的名称, 后续可根据该名称进行查看
$ auditctl -w /etc/ssh/sshd_config -p rwa -k sshd_config

# 取消监控
$ auditctl -W /etc/ssh/sshd_config -p rwa -k sshd_config
```


# 查看监控数据

查看监控数据使用 `ausearch` 命令：

```bash
# 查看监控数据
#   -k: 规则名称
$ ausearch -k sshd_config
```

# 结果

根据audit的监控时间，可以很明确的查看到监控对象在什么时间点被那个进程访问、修改。

