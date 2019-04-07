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

`tcmalloc`采用分层的设计，其内存对象被划分为*Small*、*Large* 2个等级，每个等级的对象占用内存各不相同，每个线程都有自己的本地缓存（谷歌出品比属精品）。

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

##### 分配

![](https://ws1.sinaimg.cn/large/006tNc79gy1g1u3ru12tvj318c0rctdk.jpg)

#### Large

大于`32Kb`的对象

![](https://ws3.sinaimg.cn/large/006tNc79ly1g1t0wqfxtkj311c0lamyd.jpg)

一共有256中，`[1，255]`类型每一项分别占用`n*4k`，`rest`为特殊项。

##### 分配

- [1，255]

```shell
1：根据Object Size查找适合的`k pages`
2：依次查找`last free list entry`，如果查不到`free list entry`，即无空闲内存，则向操作系统重新申请`k*4k`内存，作为`free entry`链接到`last free list`
```

> 内存申请可以使用`(using sbrk, mmap, or by mapping in portions of /dev/mem`

- 256

和前面的步骤类似.

1）大对象的分配由**Central heap**负责；2）Requested size is rounded up to number of pages(4kB)

> be round up to 向上取整



### Span

Manages memory in units called `Span`

- Runs of **contiguous** memory pages
- Metadata is **kept separated from the allocation arena**

##### 分配流程

直接从`Central Heap`分配，以`页`来适配，比如，对象的大小大于一页，则分配2页的空间。

### 层次

#### Thread Cache

#### CentralCache

#### PageHeap



# Go Memory

## Interview

```go
package main

func main(){
  f()
}

//go:noinline
func f() *int {
  i := 10
  return &i
}
```

**注：** 关闭内敛编译，否则经过编译器优化后

```go
package main

func main(){
  i := 10
}
```

编译：`go build -gcflags "-m -m" alloc.go`

```sh
# command-line-arguments
./alloc.go:8:6: cannot inline f: marked go:noinline
./alloc.go:3:6: cannot inline main: function too complex: cost 82 exceeds budget 80
./alloc.go:10:9: &i escapes to heap
./alloc.go:10:9: 	from ~r0 (return) at ./alloc.go:10:2
./alloc.go:9:2: moved to heap: i
```

关键点：

- &i escapes to heap

逃逸到堆

- move to heap: i

*i* 迁移到堆(作为指针返回)

查看一下反汇编的情况：

```go
$ go tool compile -S main.go

"".f STEXT size=79 args=0x8 locals=0x18
	0x0000 00000 (alloc.go:8)	TEXT	"".f(SB), $24-8
	0x0000 00000 (alloc.go:8)	MOVQ	(TLS), CX
	0x0009 00009 (alloc.go:8)	CMPQ	SP, 16(CX)
	0x000d 00013 (alloc.go:8)	JLS	72
	0x000f 00015 (alloc.go:8)	SUBQ	$24, SP
	0x0013 00019 (alloc.go:8)	MOVQ	BP, 16(SP)
	0x0018 00024 (alloc.go:8)	LEAQ	16(SP), BP
	0x001d 00029 (alloc.go:8)	FUNCDATA	$0, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
	0x001d 00029 (alloc.go:8)	FUNCDATA	$1, gclocals·69c1753bd5f81501d95132d08af04464(SB)
	0x001d 00029 (alloc.go:8)	FUNCDATA	$3, gclocals·9fb7f0986f647f17cb53dda1484e0f7a(SB)
	0x001d 00029 (alloc.go:9)	PCDATA	$2, $1
	0x001d 00029 (alloc.go:9)	PCDATA	$0, $0
	0x001d 00029 (alloc.go:9)	LEAQ	type.int(SB), AX
	0x0024 00036 (alloc.go:9)	PCDATA	$2, $0
	0x0024 00036 (alloc.go:9)	MOVQ	AX, (SP)
	0x0028 00040 (alloc.go:9)	CALL	runtime.newobject(SB)
```

**runtime.newobject**申请对象

## 层级

和 `tcmalloc` 不同，go memory分为3个level，分别为`tiny, small, large`

- Tiny 

小于等于16字节(no pointer)

- small

小于等于32字节大于16字节

- Large

大于32字节

### Go 分配器

go的垃圾回收是`并行`的，是不是全部步骤都是` 并行`？

- 检索所有对象(并行)
- 标记哪些对象是active的(并行)
- 交换未active的对象(stop the world)

### Small Allocator

![](https://ws1.sinaimg.cn/large/006tNc79gy1g1u5lyie1dj30rq0rs410.jpg)

>  At source-code: sizeclasses.go

`66`种规格小对象，具体意思如`class1`：

- byets/object 

每个对象的代销不超过8字节

- bytes/span

跨度为8192 kB = 8 kB

- objects

可分配的对象，计算过程：(8 kB / 8B) = 1k = 1024

即每个`span`管理`1024`个这种类型的对象

源码中有几个关键的常量：

```go
const (
	_MaxSmallSize   = 32768 // 小对象的最大值
	smallSizeDiv    = 8
	smallSizeMax    = 1024
	largeSizeDiv    = 128
	_NumSizeClasses = 67
	_PageShift      = 13
)
```



```go
var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536, 1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}
var class_to_allocnpages = [_NumSizeClasses]uint8{0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 2, 1, 2, 1, 2, 1, 3, 2, 3, 1, 3, 2, 3, 4, 5, 6, 1, 7, 6, 5, 4, 3, 5, 7, 2, 9, 7, 5, 8, 3, 10, 7, 4}

```



[Memory Allocation-Luis Ceze]: https://youtu.be/RSuZhdwvNmA

## 