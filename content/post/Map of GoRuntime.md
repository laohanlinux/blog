---
title: "Map of GoRuntime"
date: 2019-04-05T23:17:15+08:00
tags:
- Map
- GoRuntime
categories: 
- GoKernel
- Analysis of Source
draft: true
---

# View

GoMap实际上就是一个hashTable，数据存储在数组buckets中。每个bucket包含 8 个键值对。hash的低8位用于映射bucket，在bucket内，使用hash的剩下的高位用于区分。

如果超过8个键被hash到某个bucket，需要连接到额外的buckets。



![](https://ws4.sinaimg.cn/large/006tNc79gy1g1wpiggsrhj310q0s40y4.jpg)

