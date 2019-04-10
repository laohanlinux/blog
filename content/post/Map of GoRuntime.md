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

<center>
  <img src="https://ws4.sinaimg.cn/large/006tNc79gy1g1wpiggsrhj310q0s40y4.jpg">
</center>

hmap.B 可容纳的键值对: *2^B*；hmap.bucketSize: 每个桶的大小；hmap.buckets: *2^B* Buckets的数组；hmap.oldbuckets: 旧桶，在迁移时不为空；nevacuate：迁移进度；extra：？

比如有这么一个hashmap：

<center>
  <img src = "https://ws1.sinaimg.cn/large/006tNc79gy1g1xn5eruszj30u00x1n0u.jpg">
</center>



<center>
  <img src = "https://ws4.sinaimg.cn/large/006tNc79gy1g1wxb61f09j30ae15g40q.jpg" with = 750, hight = 800>
</center>

# Access map[key]

<center>
  <img src = "https://ws2.sinaimg.cn/large/006tNc79gy1g1xnd8mli2j3074118mxx.jpg">
</center>

- 计算桶的位置

````go
alg := t.key.alg
hash := alg.hash(key, uintptr(h.hash0))
m := bucketMask(h.B) //
b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
````

- 检查是否处于oldBuckets

```go
if c := h.oldbuckets; c != nil {
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
			m >>= 1
		}
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		if !evacuated(oldb) {
			b = oldb
		}
	}
```



# Dilatation

既然是hashTable，当数据量大的时候，检索会越来越慢，该如何解决这些问题。Go的Map采用了传统的扩容方式，如下：

![](https://ws2.sinaimg.cn/large/006tNc79gy1g1wujqajuaj31is0poq7l.jpg)

即每次扩容，hashTable Bucket以两倍的方式进行扩容，扩容后，理论上来说，在检索某个值的时候，路径变短了。 

这时候同时出现了两个`map`，旧map的键值对会逐步迁移至新的map，为了性能的考虑，不会一次性迁移，采用分批次的策略，那么就会需要解决如下的问题：

- 迁移的时机，什么操作会触发迁移
- 每次的迁移数据量是多少

除此之外，还有一个比较有意思的问题，键值对达到多少时会触发扩容？在hashTable中有个*loadFactor*的概念，中文意思是*加载因子*或者叫做*负载系数*。

源码中是这么解析的：负载系数太大就会有很多溢出的桶(buckets)，如果太小就会浪费太多空间。然后作者就给出了一份测试数据：

```
//  loadFactor    %overflow  bytes/entry     hitprobe    missprobe
//        4.00         2.13        20.77         3.00         4.00
//        4.50         4.05        17.30         3.25         4.50
//        5.00         6.85        14.77         3.50         5.00
//        5.50        10.55        12.94         3.75         5.50
//        6.00        15.27        11.67         4.00         6.00
//        6.50        20.90        10.79         4.25         6.50
//        7.00        27.14        10.15         4.50         7.00
//        7.50        34.03         9.73         4.75         7.50
//        8.00        41.10         9.40         5.00         8.00
//
// %overflow   = percentage of buckets which have an overflow bucket
// bytes/entry = overhead bytes used per key/value pair
// hitprobe    = # of entries to check when looking up a present key
// missprobe   = # of entries to check when looking up an absent key
//
```

资料扩展：

[google hashmap load factor](<https://www.google.com/search?q=hashmap+load+factor&spell=1&sa=X&ved=0ahUKEwi7haG6u8PhAhWaHzQIHf7EAN0QBQgpKAA&biw=1680&bih=916>)

[wiki](<https://en.wikipedia.org/wiki/Hash_table>)

# Other Special Function or Value

## hashFunction

TODO

## ToHash[0]

桶的迁移进度状态

- Empty

槽为空，即该桶无元素

<center>
  <img src = "https://ws3.sinaimg.cn/large/006tNc79gy1g1xy3qxxysj303s07ut8m.jpg">
</center>

- evacuatedEmpty = 1

槽为空，并且桶被迁移到新的地方

<center>
  <img src = "https://ws3.sinaimg.cn/large/006tNc79gy1g1xy4ra3o6j30a60gkt9e.jpg">
</center>

- evacuatedX

键值对合法，键值对被迁移到新表的**前半位置**

<center>
  <img src = "https://ws3.sinaimg.cn/large/006tNc79gy1g1xy5tz8i9j30a60gk0th.jpg">
</center>

- evacuatedY

键值对合法，键值对被迁移到新表的**后半位置

<center>
  <img src = "https://ws4.sinaimg.cn/large/006tNc79gy1g1xy5fpg75j30a60gkq3p.jpg">
</center>



- minTopHash

## bucketShift()

```go
// bucketShift returns 1<<b, optimized for code generation.
func bucketShift(b uint8) uintptr {
	if sys.GoarchAmd64|sys.GoarchAmd64p32|sys.Goarch386 != 0 {
		b &= sys.PtrSize*8 - 1 // help x86 archs remove shift overflow checks
		// 4 * 8 - 1 = 31
		// b &= 31 = b &= 1111 1 即b的取值范围[0, 1111 1]，再看下面可知，被限制在 1<<31内
		// 在[sys.GoarchAmd64|sys.GoarchAmd64p32|sys.Goarch386]，这几种架构中被限制
	}
	return uintptr(1) << b
}
```

[深入理解 Go map：初始化和访问元素](<https://github.com/EDDYCJY/blog/blob/master/golang/pkg/2019-03-04-%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Go-map-%E5%88%9D%E5%A7%8B%E5%8C%96%E5%92%8C%E8%AE%BF%E9%97%AE%E5%85%83%E7%B4%A0.md>)

