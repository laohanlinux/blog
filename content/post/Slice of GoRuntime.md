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

![](https://ws3.sinaimg.cn/large/006tNc79gy1g1w7kwcyk1j30ci0723yv.jpg)

- array

底层指向数组的指针，是一块连续的空间

- len

大小

- cap

容量

### New slice

![image-20190409112312727](https://ws4.sinaimg.cn/large/006tNc79gy1g1w7vakto3j30ci0jkgm5.jpg)

[mallocgc](<https://laohanlinux.github.io/2019/02/06/memory-manager-of-goruntime/>)

### 扩容

slice比较有意思的是扩容阶段，slice到底是以怎样的方式进行扩容的呢？

主要逻辑在`growslice`函数

### Example

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

![image-20190409120804468](https://ws1.sinaimg.cn/large/006tNc79gy1g1w95y8de0j31nu0p4dkp.jpg)

![](https://ws2.sinaimg.cn/large/006tNc79gy1g1w9a8z2waj308c06omxf.jpg)

