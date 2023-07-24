---
title: "EBPF 随笔"
date: 2023-07-24T14:22:52+08:00
draft: false
tags:
  - EBPF
---

记录关于EBPF的随笔，以Linux-5.10版本的内核为基准。

<!--more-->

# 基础

## 支持的EBPF程序类型

Kernel 支持的EBPF程序类型[源代码](https://github.com/torvalds/linux/blob/v5.10/include/uapi/linux/bpf.h#L170):

| 程序类型                              | 描述                                                         | 备注                            |
| ------------------------------------- | ------------------------------------------------------------ | ------------------------------- |
| BPF_PROG_TYPE_SOCKET_FILTER           | 对流量进行复制、过滤、统计等操作，Hook 位置`sock_queue_rcv_skb` | <font color="blue">Socket</font> |
| BPF_PROG_TYPE_KPROBE                  | 通过 `kprobe`/`kretprobe` 观测内核函数。 `k[ret]probe_perf_func()` 会执行加载到 probe 点的 BPF 程序。 |                                 |
| BPF_PROG_TYPE_SCHED_CLS               | 分类器:`tc classifier`，将BPF程序作为`classifiers`加载到`ingress/egress` `Hook`点：`sch_handle_ingress`、`sch_handle_egress` |                                 |
| BPF_PROG_TYPE_SCHED_ACT               | 动作:`tc action`，将BPF程序作为`actions`加载到`ingress/egress` `Hook`点 ：`sch_handle_ingress`、`sch_handle_egress` |                                 |
| BPF_PROG_TYPE_TRACEPOINT              | `perf_trace_<event_class>()` [源代码](https://github.com/torvalds/linux/blob/v5.10/include/trace/perf.h) |                                 |
| BPF_PROG_TYPE_XDP                     | `XDP`位于设备驱动层，有网卡/驱动及对应内核版本支持；[BCC-XDP](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md#xdp)，主要用于防火墙、负载均衡等场景；对于没有实现XDP的驱动，内核也实现了`Generic XDP`的实现，见[net/core/dev.c](https://github.com/torvalds/linux/blob/v5.10/net/core/dev.c) |                                 |
| BPF_PROG_TYPE_PERF_EVENT              |                                                              |                                 |
| BPF_PROG_TYPE_CGROUP_SKB              | 在`CGroup`级别`IP Ingress/Egress`放行/丢弃数据包，Attach类型：`BPF_CGROUP_INET_INGRESS`、`BPF_CGROUP_INET_EGRESS`；Hook位置：`__sk_receive_skb`、`tcp_v4_rcv->tcp_filter`、`udp_queue_rcv_one_skb` ==> `sk_filter_trim_cap`；出方向：`ip[6]_finish_output` |                                 |
| BPF_PROG_TYPE_CGROUP_SOCK             | 在`inet_create`中执行，可允许/拒绝网络访问，`BPF_CGROUP_INET_SOCK_CREATE`、`BPF_CGROUP_SOCK_OPS`；整个连接的生命周期只调用一次？[源代码](https://github.com/torvalds/linux/blob/v5.10/kernel/bpf/syscall.c#L1990) | <font color="blue">Socket</font> |
| BPF_PROG_TYPE_LWT_IN                  |                                                              |                                 |
| BPF_PROG_TYPE_LWT_OUT                 |                                                              |                                 |
| BPF_PROG_TYPE_LWT_XMIT                |                                                              |                                 |
| BPF_PROG_TYPE_SOCK_OPS                | 依赖`CGroup/v2`，可以指定` BPF_CGROUP_SOCK_OPS`类型，将BPF程序`Attach`到`CGroup`的文件描述符上。 | <font color="blue">Socket</font> |
| BPF_PROG_TYPE_SK_SKB                  | 通过修改skb/socket信息，可用作socket重定向，Hook位置`smap_parse_func_strparser`、`smap_verdict_func` | <font color="blue">Socket</font> |
| BPF_PROG_TYPE_CGROUP_DEVICE           |                                                              |                                 |
| BPF_PROG_TYPE_SK_MSG                  |                                                              | <font color="blue">Socket</font> |
| BPF_PROG_TYPE_RAW_TRACEPOINT          |                                                              |                                 |
| BPF_PROG_TYPE_CGROUP_SOCK_ADDR        | 可以操作指定CGroup控制的用户程序的IP地址及端口号，`cgroup/connect[46]`等函数；[源代码](https://github.com/torvalds/linux/blob/v5.10/kernel/bpf/syscall.c#L2000) | <font color="blue">Socket</font> |
| BPF_PROG_TYPE_LWT_SEG6LOCAL           |                                                              |                                 |
| BPF_PROG_TYPE_LIRC_MODE2              |                                                              |                                 |
| BPF_PROG_TYPE_SK_REUSEPORT            | 可以使用多个socket侦听在相同的端口地址，有助于新老系统的业务切换；需Hook `CGroup`级别的`socket`事件 | <font color="blue">Socket</font> |
| BPF_PROG_TYPE_FLOW_DISSECTOR          |                                                              |                                 |
| BPF_PROG_TYPE_CGROUP_SYSCTL           |                                                              |                                 |
| BPF_PROG_TYPE_RAW_TRACEPOINT_WRITABLE |                                                              |                                 |
| BPF_PROG_TYPE_CGROUP_SOCKOPT          |                                                              |                                 |
| BPF_PROG_TYPE_TRACING                 |                                                              |                                 |
| BPF_PROG_TYPE_STRUCT_OPS              |                                                              |                                 |
| BPF_PROG_TYPE_EXT                     |                                                              |                                 |
| BPF_PROG_TYPE_LSM                     |                                                              |                                 |
| BPF_PROG_TYPE_SK_LOOKUP               |                                                              | <font color="blue">Socket</font> |

## 支持的EBPF-Attach类型

Kernel 支持的EBPF Attach类型[源代码](https://github.com/torvalds/linux/blob/v5.10/include/uapi/linux/bpf.h#L204):



| Attach 类型                  | 描述                           | 备注                                                         |
| ---------------------------- | ------------------------------ | ------------------------------------------------------------ |
| BPF_CGROUP_INET_INGRESS      | BPF_PROG_TYPE_CGROUP_SKB       | <font color="blue">CGroup</font> [源代码](https://github.com/torvalds/linux/blob/v5.10/kernel/bpf/cgroup.c) |
| BPF_CGROUP_INET_EGRESS       | BPF_PROG_TYPE_CGROUP_SKB       | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_INET_SOCK_CREATE  | BPF_PROG_TYPE_CGROUP_SOCK      | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_SOCK_OPS          | BPF_PROG_TYPE_SOCK_OPS         | <font color="blue">CGroup</font>                             |
| BPF_SK_SKB_STREAM_PARSER     | BPF_PROG_TYPE_SK_SKB           |                                                              |
| BPF_SK_SKB_STREAM_VERDICT    | BPF_PROG_TYPE_SK_SKB           |                                                              |
| BPF_CGROUP_DEVICE            | BPF_PROG_TYPE_CGROUP_DEVICE    | <font color="blue">CGroup</font>                             |
| BPF_SK_MSG_VERDICT           | BPF_PROG_TYPE_SK_MSG           |                                                              |
| BPF_CGROUP_INET4_BIND        | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_INET6_BIND        | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_INET4_CONNECT     | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_INET6_CONNECT     | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_INET4_POST_BIND   | BPF_PROG_TYPE_CGROUP_SOCK      | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_INET6_POST_BIND   | BPF_PROG_TYPE_CGROUP_SOCK      | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_UDP4_SENDMSG      | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_UDP6_SENDMSG      | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_LIRC_MODE2               | BPF_PROG_TYPE_LIRC_MODE2       |                                                              |
| BPF_FLOW_DISSECTOR           | BPF_PROG_TYPE_FLOW_DISSECTOR   |                                                              |
| BPF_CGROUP_SYSCTL            | BPF_PROG_TYPE_CGROUP_SYSCTL    | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_UDP4_RECVMSG      | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_UDP6_RECVMSG      | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_GETSOCKOPT        | BPF_PROG_TYPE_CGROUP_SOCKOPT   | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_SETSOCKOPT        | BPF_PROG_TYPE_CGROUP_SOCKOPT   | <font color="blue">CGroup</font>                             |
| BPF_TRACE_RAW_TP             |                                |                                                              |
| BPF_TRACE_FENTRY             |                                |                                                              |
| BPF_TRACE_FEXIT              |                                |                                                              |
| BPF_MODIFY_RETURN            |                                |                                                              |
| BPF_LSM_MAC                  |                                |                                                              |
| BPF_TRACE_ITER               | BPF_PROG_TYPE_TRACING          |                                                              |
| BPF_CGROUP_INET4_GETPEERNAME | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_INET6_GETPEERNAME | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_INET4_GETSOCKNAME | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_CGROUP_INET6_GETSOCKNAME | BPF_PROG_TYPE_CGROUP_SOCK_ADDR | <font color="blue">CGroup</font>                             |
| BPF_XDP_DEVMAP               |                                |                                                              |
| BPF_CGROUP_INET_SOCK_RELEASE | BPF_PROG_TYPE_CGROUP_SOCK      | <font color="blue">CGroup</font>                             |
| BPF_XDP_CPUMAP               |                                |                                                              |
| BPF_SK_LOOKUP                | BPF_PROG_TYPE_SK_LOOKUP        |                                                              |
| BPF_XDP                      | BPF_PROG_TYPE_XDP              |                                                              |

