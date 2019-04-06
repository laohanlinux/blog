---
title: "Memory Manager of GoRuntime"
date: 2019-02-06T11:49:52+08:00
draft: true
---

## 前言

`Go`内存管理是`runtime`比较重要的一部分，`Go`内存管理算法来至于`TCMalloc`，非常类似。`tcmalloc`已经发展好长一段时间了，是非常高效的一种内存管理算法，下面简单聊一下`tcmalloc`。

### 虚拟内存

![](https://ws1.sinaimg.cn/large/006tNc79ly1g1sul0k0sbj311e0twq5z.jpg)

在现代系统中，保护模式下，进程看到的内存地址都是`虚拟地址`，可以通过页表进行转换，转换到实际的主存RAW或者磁盘中。

在go中，对象的地址在编译的时候就确定，来验证一下.

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	rand.Seed(time.Now().UnixNano())

	i := rand.Intn(100)

	fmt.Printf("%v at %p\n", i, &i)
}
```

Run:

```
└─[$]> ./virtual_addr
5 at 0xc0000160c0
└─[$]> ./virtual_addr
27 at 0xc0000160c0
└─[$]> ./virtual_addr
60 at 0xc0000160c0
```

多次运行，对象i的地址都是一致的

## TCMalloc

`tcmalloc`采用分层的设计，其内存对象被划分为`Small`、`Medium`、`Large`三个等级，每个等级的对象占用内存各不相同，每个线程都有自己的本地缓存（谷歌出品比属精品）。

结构:

![](https://ws1.sinaimg.cn/large/006tNc79ly1g1szxe9enmj30qg0ii0ul.jpg)

### 内存分类

#### Small

小于等于`32Kb`的对象

![](https://ws2.sinaimg.cn/large/006tNc79ly1g1t0i0ojahj30q40dkmxq.jpg)

小对象被映射成`170`种规格的大小的小对象，每个的规格不一定相同，但满足`2^N Bytes`，如：

```shell
class0 = `2^3 Bytes` = `8Bytes`
class1 = `2^4 Bytes` = `16Bytes`
...
class2 = `32 * 2^10` = `32Bytes`
```





#### Large

大于`32Kb`的对象

##### 分配流程

直接从`Central Heap`分配，以`页`来适配，比如，对象的大小大于一页，则分配2页的空间。

### 层次

#### Thread Cache

#### CentralCache

#### PageHeap

[Memory Allocation-Luis Ceze]: https://youtu.be/RSuZhdwvNmA

## 