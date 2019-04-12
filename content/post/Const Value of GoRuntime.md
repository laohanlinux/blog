---
title: "Const Value of GoRuntime"
date: 2019-04-10T01:32:19+08:00
tags:
- Const Value
- GoRuntime
categories: 
- GoKernel
- Analysis of Source
draft: true
---

最近在看Go Runtime源码的时候，发现很多字面量，比较有意思。

- PtrSize = 4

```go
const PtrSize = 4 << (^uintptr(0) >> 64)
// unsafe.Sizeof(uintptr(0)) but an ideal const
```

## Map

- bucketCntBits = 3

桶个数占位数

- bucketCnt = 1 << bucketCntBits = 8

桶个数

- loadFactor = 6.5

触发扩容操作的最大装载因子的临界值

- maxKeySize = 128 
- maxValueSize = 128

为了保持内联，键值对的最大长度都是`128`字节，如果超过`128`个字节，就存储它的指针

- dataOffset = unsafe.Offsetof(struct {b bmap; v int64} {}.v)

`dataOffset`应该是`bmap struct`的大小，但是需要内存字节

- noCheck = 1<<(8*sys.PtrSize) - 1

用于迭代检查的哨兵桶ID



