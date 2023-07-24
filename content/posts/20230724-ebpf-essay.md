---
title: "EBPF 随笔"
date: 2023-07-24T14:22:52+08:00
draft: true
tags:
  - EBPF
---

记录关于EBPF的随笔，以Linux-5.10版本的内核为基准。

<!--more-->

# 基础

## 支持的EBPF程序类型

Kernel 支持的EBPF程序类型[源代码](https://github.com/torvalds/linux/blob/v5.10/include/uapi/linux/bpf.h#L170):

| 程序类型                              | 描述 |      |
| ------------------------------------- | ---- | ---- |
| BPF_PROG_TYPE_SOCKET_FILTER           |      |      |
| BPF_PROG_TYPE_KPROBE                  |      |      |
| BPF_PROG_TYPE_SCHED_CLS               |      |      |
| BPF_PROG_TYPE_SCHED_ACT               |      |      |
| BPF_PROG_TYPE_TRACEPOINT              |      |      |
| BPF_PROG_TYPE_XDP                     |      |      |
| BPF_PROG_TYPE_PERF_EVENT              |      |      |
| BPF_PROG_TYPE_CGROUP_SKB              |      |      |
| BPF_PROG_TYPE_CGROUP_SOCK             |      |      |
| BPF_PROG_TYPE_LWT_IN                  |      |      |
| BPF_PROG_TYPE_LWT_OUT                 |      |      |
| BPF_PROG_TYPE_LWT_XMIT                |      |      |
| BPF_PROG_TYPE_SOCK_OPS                |      |      |
| BPF_PROG_TYPE_SK_SKB                  |      |      |
| BPF_PROG_TYPE_CGROUP_DEVICE           |      |      |
| BPF_PROG_TYPE_SK_MSG                  |      |      |
| BPF_PROG_TYPE_RAW_TRACEPOINT          |      |      |
| BPF_PROG_TYPE_CGROUP_SOCK_ADDR        |      |      |
| BPF_PROG_TYPE_LWT_SEG6LOCAL           |      |      |
| BPF_PROG_TYPE_LIRC_MODE2              |      |      |
| BPF_PROG_TYPE_SK_REUSEPORT            |      |      |
| BPF_PROG_TYPE_FLOW_DISSECTOR          |      |      |
| BPF_PROG_TYPE_CGROUP_SYSCTL           |      |      |
| BPF_PROG_TYPE_RAW_TRACEPOINT_WRITABLE |      |      |
| BPF_PROG_TYPE_CGROUP_SOCKOPT          |      |      |
| BPF_PROG_TYPE_TRACING                 |      |      |
| BPF_PROG_TYPE_STRUCT_OPS              |      |      |
| BPF_PROG_TYPE_EXT                     |      |      |
| BPF_PROG_TYPE_LSM                     |      |      |
| BPF_PROG_TYPE_SK_LOOKUP               |      |      |

## 支持的EBPF-Attach类型

Kernel 支持的EBPF Attach类型[源代码](https://github.com/torvalds/linux/blob/v5.10/include/uapi/linux/bpf.h#L204):



| Attach 类型                  | 描述 |      |
| ---------------------------- | ---- | ---- |
| BPF_CGROUP_INET_INGRESS      |      |      |
| BPF_CGROUP_INET_EGRESS       |      |      |
| BPF_CGROUP_INET_SOCK_CREATE  |      |      |
| BPF_CGROUP_SOCK_OPS          |      |      |
| BPF_SK_SKB_STREAM_PARSER     |      |      |
| BPF_SK_SKB_STREAM_VERDICT    |      |      |
| BPF_CGROUP_DEVICE            |      |      |
| BPF_SK_MSG_VERDICT           |      |      |
| BPF_CGROUP_INET4_BIND        |      |      |
| BPF_CGROUP_INET6_BIND        |      |      |
| BPF_CGROUP_INET4_CONNECT     |      |      |
| BPF_CGROUP_INET6_CONNECT     |      |      |
| BPF_CGROUP_INET4_POST_BIND   |      |      |
| BPF_CGROUP_INET6_POST_BIND   |      |      |
| BPF_CGROUP_UDP4_SENDMSG      |      |      |
| BPF_CGROUP_UDP6_SENDMSG      |      |      |
| BPF_LIRC_MODE2               |      |      |
| BPF_FLOW_DISSECTOR           |      |      |
| BPF_CGROUP_SYSCTL            |      |      |
| BPF_CGROUP_UDP4_RECVMSG      |      |      |
| BPF_CGROUP_UDP6_RECVMSG      |      |      |
| BPF_CGROUP_GETSOCKOPT        |      |      |
| BPF_CGROUP_SETSOCKOPT        |      |      |
| BPF_TRACE_RAW_TP             |      |      |
| BPF_TRACE_FENTRY             |      |      |
| BPF_TRACE_FEXIT              |      |      |
| BPF_MODIFY_RETURN            |      |      |
| BPF_LSM_MAC                  |      |      |
| BPF_TRACE_ITER               |      |      |
| BPF_CGROUP_INET4_GETPEERNAME |      |      |
| BPF_CGROUP_INET6_GETPEERNAME |      |      |
| BPF_CGROUP_INET4_GETSOCKNAME |      |      |
| BPF_CGROUP_INET6_GETSOCKNAME |      |      |
| BPF_XDP_DEVMAP               |      |      |
| BPF_CGROUP_INET_SOCK_RELEASE |      |      |
| BPF_XDP_CPUMAP               |      |      |
| BPF_SK_LOOKUP                |      |      |
| BPF_XDP                      |      |      |

