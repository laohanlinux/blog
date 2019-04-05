---
title: "Go Memory Manager"
date: 2019-04-04T17:12:18+08:00
notoc: false
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

