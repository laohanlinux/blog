---
title: "Slice of GoRuntime"
date: 2019-04-05T23:17:07+08:00
tags:
- Slice
- GoRuntime
categories: 
- GoKernel
- Analysis of Source
draft: true
---

## Overview

![](https://i.loli.net/2019/05/03/5ccbe0c58f1cc.jpg)

- array

底层指向数组的指针，是一块连续的空间

- len

大小

- cap

容量

## New slice

![image-20190409112312727](https://i.loli.net/2019/05/03/5ccc041ed25a1.jpg)

[mallocgc](<https://laohanlinux.github.io/2019/02/06/memory-manager-of-goruntime/>)

## Dilatation

slice比较有意思的是扩容阶段，slice到底是以怎样的方式进行扩容的呢？

主要逻辑在`growslice`函数

![](https://i.loli.net/2019/05/03/5ccbe0c64ae6d.jpg)

*注：* 主要关键的两个逻辑：所需容量小于1024，2^N；大于1024，1/4缓慢递增，直至能容纳所需容量的大小

## Example

往上看到了一个比较有意思的例子

```go
package main

import "fmt"

func main() {
	s := []int{5}
	fmt.Println("s1:", len(s), cap(s))
	s = append(s, 7)
	fmt.Println("s2:", len(s), cap(s))
	s = append(s, 9)
	fmt.Println("s3:", len(s), cap(s))
	x := append(s, 11)
	fmt.Println("x-->", x)
	fmt.Println("sx:", len(s), cap(s))
	fmt.Println("x:", len(x), cap(x))
	y := append(s, 12)
	fmt.Println("sy:", len(s), cap(s))
	fmt.Println("y-->", y)
	fmt.Println(s, x, y)
  fmt.Printf("s:%p, x:%p, y:%p\n", s, x, y)
}
```

输出：

```shell
s1: 1 1
s2: 2 2
s3: 3 4
x--> [5 7 9 11]
sx: 3 4
x: 4 4
sy: 3 4
y--> [5 7 9 12]
[5 7 9] [5 7 9 12] [5 7 9 12]
s:0xc000018200, x:0xc000018200, y:0xc000018200
```

Why?

![image-20190409120804468](https://i.loli.net/2019/05/03/5ccbe0ab0eed6.jpg)

![](https://i.loli.net/2019/05/03/5ccbe0b9bbc9e.jpg)

[深入理解 Go Slice](<https://github.com/EDDYCJY/blog/blob/master/golang/pkg/2018-12-11-%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Go-Slice.md>)

