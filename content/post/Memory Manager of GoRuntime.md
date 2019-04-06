---
title: "Memory Manager of GoRuntime"
date: 2019-02-06T11:49:52+08:00
draft: true
---

## 前言

`Go`内存管理是`runtime`比较重要的一部分，`Go`内存管理算法来至于`TCMalloc`，非常类似。`tcmalloc`已经发展好长一段时间了，是非常高效的一种内存管理算法，下面简单聊一下`tcmalloc`。

## TCMalloc

`tcmalloc`采用分层的设计，其内存对象被划分为`Small`、`Medium`、`Large`三个等级，每个等级的对象占用内存各不相同。

### Memory Level

#### Small

#### Medium

#### Large

### 层次

#### Thread Cache

#### CentralCache

#### PageHeap



[Memory Allocation-Luis Ceze]: https://youtu.be/RSuZhdwvNmA

## Enter

### 虚拟内存

![](https://ws1.sinaimg.cn/large/006tNc79ly1g1sul0k0sbj311e0twq5z.jpg)

在现代系统中，保护模式下，进程看到的内存地址都是`虚拟地址`，可以通过页表进行转换，转换到实际的主存RAW或者磁盘中。

