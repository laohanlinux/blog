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

- bucketCntBits = 3

桶个数占位数

- bucketCnt = 1 << bucketCntBits = 8

桶个数

