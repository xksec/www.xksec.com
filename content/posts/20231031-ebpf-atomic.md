---
title: "EBPF原子操作避坑指南"
date: 2023-10-31T22:13:51+08:00
draft: false
tags:
  - EBPF
---

好久没更新了，有点惭愧。且最近在EBPF的原子性上也头疼了将近一周的时间，主要是通过测试不同的原子性方法，来兼容不同的环境。

这次将测试验证的过程记录下来，以避免以后踩到同样类型的坑里。

<!--more-->

# 环境信息

手里有三种环境见下表，为了让一套代码同时支持三种不同的环境，反复做了很多次试验，这里只记录成功的结果。

| 操作系统     | kernel 版本 | clang & LLVM 版本 |
| ------------ | ----------- | ----------------- |
| CentOS 7.9   | 5.10.0-1    | 9.0.1             |
| CentOS 7.9   | 4.18.0-193  | 9.0.1             |
| Ubuntu 23.04 | 6.2.0-35    | 15.0.7            |

# 原子性机制

本来是知道EBPF支持有限的锁的机制，但是考虑到以前经常从kernel中复制汇编代码，进行CAS(compare and swap)操作。

这里也是想优先使用一句话的原子CAS操作，来解决EBPF在多CPU访问一个Entry的互斥问题。

## SYNC系列

首先想到的是无锁化的原子操作，结果从其他地方复制过来的CAS汇编代码，在LLVM中均无法编译通过。

这里又想到了GCC及LLVM内建的__sync_系列函数：
[ [GCC内建](https://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Atomic-Builtins.html) [LLVM内建](https://llvm.org/docs/Atomics.html) ]
```c
static __u64 data = 10086;
__u64 old = __sync_val_compare_and_swap(&data, 10086, 10010);
bpf_printk("old: %d",old);
```
结果在6.2的环境中可以正常运行，在4.18及5.10环境中报错`Cannot select: 0x55c89940f1e8: i64,ch = AtomicCmpSwap<(load store seq_cst seq_cst 8 on @data)>`。遂放弃该方法。


## SYNC系列②

通过查阅LLVM的文档，发现`__sync_val_compare_and_swap`该函数是LLVM在后边的版本才添加支持的。那么退而求其次，使用`fetch-add`函数可以模拟原子锁的操作：

```c
static __u64 data = 10086;
__u64 old = __sync_fetch_and_add(&data, 1);
bpf_printk("old: %d", old);
```
这次可以编译通过了，且在 6.2 及5.10上正常运行，但是在4.18系统上加载报错：`invalid argument: BPF_STX uses reserved fields`，及`stx`指令访问了保留的字段。显而易见4.18版本还不支持`fetch-add`类的操作。遂放弃。

## bpf_spin_lock

又回到了起点，EBPF是支持`bpf_spin_lock`操作的，那么只好使用此类函数实现原子性了：
```c
static __u64 data = 10086;
static struct bpf_spin_lock lock;
bpf_spin_lock(&lock);
data++;
bpf_spin_unlock(&lock);
bpf_printk("after unlock: %d", data);
```

结果在6.20及5.10上可以正常运行，在4.18上加载报错：`reference to "lock" in section SHN_COMMON: not supported`。遂放弃。

## bpf_spin_lock②

通过查看ebpf的文档，发现在低版本的kernel中，`bpf_spin_lock`只支持在`BPF_MAP_TYPE_HASH`和`BPF_MAP_TYPE_ARRAY`的Value中实现。
经过反复验证，甚至到6.2的kernel也不支持`BPF_MAP_TYPE_LRU_HASH`类型的MAP。

```c
static __u64 data = 10086;

struct entry {
    struct bpf_spin_lock lock;
};

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, int);
    __type(value, struct entry);
    __uint(pinning, LIBBPF_PIN_BY_NAME);
    __uint(max_entries, 16);
    __uint(map_flags, BPF_F_NO_PREALLOC);
} map001 SEC(".maps");

void do_lock()
{   // 4.18: OK
    int k = 1;
    struct entry *e = bpf_map_lookup_elem(&map001, &k);
    if (e == NULL) {
        struct entry ee;
        memset(&ee, 0, sizeof(struct entry));
        bpf_map_update_elem(&map001, &k, &ee, BPF_NOEXIST); // 忽略返回值

        e = bpf_map_lookup_elem(&map001, &k);
    }

    if (e != NULL) {
        bpf_spin_lock(&e->lock);
        data++;
        bpf_spin_unlock(&e->lock);
        bpf_printk("after unlock: %d", data);
    }
}
```

终于，找到了一种方法，同时支持4.18、5.10、6.2的kernel。感兴趣的同学可以查看 [示例代码](https://github.com/anhk/go-ebpf-demo/blob/main/atomic/ebpf/atomic.c)

# 总结

| 内核版本                    | 4.18.0-193 | 5.10.0-1 | 6.2.0-35 |
| --------------------------- | ---------- | -------- | -------- |
| __sync_val_compare_and_swap | ❌          | ❌        | ✔️        |
| __sync_fetch_and_add        | ❌          | ✔️        | ✔️        |
| static bpf_spin_lock        | ❌          | ✔️        | ✔️        |
| MAP-bpf_spin_lock           | ✔️          | ✔️        | ✔️        |

