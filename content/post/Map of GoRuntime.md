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

![image-20190412023057466](https://ws1.sinaimg.cn/large/006tNc79gy1g1z9ces2epj30zz0u046l.jpg)

*注*：

1：为了避免字节对齐带来的空间浪费，键值对的存储并不是`键/值/键/值`这种方式，而是`键/键/值/值`的方式。

2：如果键值存储的是指针会加重gc的扫描(指针需要扫描指针指引的对象，可能会一直迭代扫描下去)，如果不是指针，那么一一迭代扫描了。

# Init Map

- Fix hint

```go
if hint < 0 || hint > int(maxSliceCap(t.bucket.size)) {
		hint = 0
}
```

- Init hashFunction

```go
if h == nil {
		h = (*hmap)(newobject(t.hmap))
	}
h.hash0 = fastrand()
```

*注：* 为了安全（加入随机因子），不停的迭代，可以看出每个new map，他们随机因子都是不同的，而且`getg().m`应该每次启动都不同的(TODO：找出更多细节).

```go
func fastrand() uint32 {
	mp := getg().m
	// Implement xorshift64+: 2 32-bit xorshift sequences added together.
	// Shift triplet [17,7,16] was calculated as indicated in Marsaglia's
	// Xorshift paper: https://www.jstatsoft.org/article/view/v008i14/xorshift.pdf
	// This generator passes the SmallCrush suite, part of TestU01 framework:
	// http://simul.iro.umontreal.ca/testu01/tu01.html
	s1, s0 := mp.fastrand[0], mp.fastrand[1]
	s1 ^= s1 << 17
	s1 = s1 ^ s0 ^ s1>>7 ^ s0>>16
	mp.fastrand[0], mp.fastrand[1] = s0, s1
	return s0 + s1
}
```

- Init B

```go
// find size parameter which will hold the requested # of elements
	B := uint8(0)
	// 计算一个合适的负载因子
	// 主要是loadFactorNum*(bucketShift(B)/loadFactorDen))
	// bucketShift(B) * loadFactorNum/loadFactorDen
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B
```

- Init hash table

```go
if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}
```

> 1:  如果B=0，则采用懒惰分配策略
>
> 2: nextOverflow: 下一个空闲的起止位置。什么意思呢？
>
> 2.1 bucekt => |bmap1|bmap2|bmap3|overflow-bmap
>
> 其中overflow-bmap为最后bmap3的溢出区overflow-bmap，那么下一调用map.newoverflow(t *maptype, b *bmap) 时相当于提前生成了nextOverflow
>
> 2.1 bucket =>  |bmap1|bmap2|bmap3
>
> 无溢出，nextOverflow自然为nil

*注*：如果初始化时，nextflow不为nil，则其布局如下:

<center>
  <img src = "https://ws2.sinaimg.cn/large/006tNc79gy1g1z9kwfft2j30jq068mxs.jpg">
</center>

- Init Memory Space

<center>
  <img src = "https://ws4.sinaimg.cn/large/006tNc79gy1g1wxb61f09j30ae15g40q.jpg" with = 750, hight = 800>
</center>

*注：* 内存分配是要涉及到页对齐，所以`nbuckets`可能会比`base`大，这时候会预分配 `nbuckets - base` 个 `nextOverflow`桶

# Lookups

```
v := m["key"] --> runtime.mapaccess1(m, "key", &v)
v, ok := m["key"] --> runtime.mapaccess2(m, "key", &v, &ok)
```

`mapaccess1`、`mapaccess2`、`mapaccess3`大同小异，分析`mapaccess1`即可。

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

- topHash

```go
// tophash calculates the tophash value for hash.
func tophash(hash uintptr) uint8 {
  // 往右移，剩下高8位，即top为hash的高8位
	top := uint8(hash >> (sys.PtrSize*8 - 8))
	if top < minTopHash {
    // 小于minTopHash时，需要加上minTopHash，[0, minToHash]已被占
 		// 用，假设top =1, top+=minTopHash = 5. 如果topHash1 = 5, 
    // 最终结果topHash = topHash1，这样也不影响后续的比较(if alg.equal(key, k))
		top += minTopHash
	}
	return top
}
```

存储高8位，简单的加速检索策略。

# Insert

```scala
m["key"] = 8120 --> runtime.mapinsert(m, "key", 8120)
```

源码函数：`mapassign(t *maptype, h *hmap, key unsafe.Pointer)`

- 计算桶位置
- 检查oldBuckets是否为空，不为空，则**增量迁移**
- 迭代桶和溢出桶，查找目标，查到则更新对应的值；否生成新的溢出桶，将新键值对保存至此，并增加计算hmap.count++

这里有些比较有意思的细节：

- key的更新-bulkBarrierPreWrite [TODO，具体细节还未知]
- 检查是否需要Grow

# Remove

```shell
delete(m, "key") --> runtime.mapdelete(m, "key")
```

- 计算桶的位置

- 检查oldBuckets是否为空，不为空，则**增量迁移**

- 迭代桶和溢出桶，查找目标

  - 查到

    1：toHash[i] = empty

    2：h.count -- 

  - 未查到

    Nothing

？TODO key/value指正和非指针处理方式稍有不同

# Iterator

```go
// A hash iteration structure.
// If you modify hiter, also change cmd/internal/gc/reflect.go to indicate
// the layout of this structure.
type hiter struct {
	key         unsafe.Pointer // Must be in first position.  Write nil to indicate iteration end (see cmd/internal/gc/range.go).
	value       unsafe.Pointer // Must be in second position (see cmd/internal/gc/range.go).
	t           *maptype
	h           *hmap
	buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
	bptr        *bmap          // current bucket
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
	startBucket uintptr        // bucket iteration started at
	offset      uint8          // intra-bucket offset to start from during iteration (should be big enough to hold bucketCnt-1)
	wrapped     bool           // already wrapped around from end of bucket array to beginning
	B           uint8
	i           uint8
	bucket      uintptr
	checkBucket uintptr
}
```

入口函数`mapiterinit(t *maptype, h *hmap, it *iter)`

- 构造迭代器对象(使用Copy方式构造)
- 选择迭代器基点
- 开始迭代

# Dilatation

既然是hashTable，当数据量大的时候，检索会越来越慢，该如何解决这些问题。Go的Map采用了传统的扩容方式，如下：

![](https://ws2.sinaimg.cn/large/006tNc79gy1g1wujqajuaj31is0poq7l.jpg)

即每次扩容，hashTable Bucket以两倍的方式进行扩容，扩容后，理论上来说，在检索某个值的时候，路径变短了。 

这时候同时出现了两个`map`，旧map的键值对会逐步迁移至新的map，为了性能的考虑，不会一次性迁移，采用*增量扩容*策略，那么就会需要解决如下的问题：

- 迁移的时机，什么操作会触发迁移

在 Go 的 mapassign 插入 Key 值、mapdelete 删除 key 值的时候都会检查当前是否在扩容中.

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

GoMap使用的是*6.5*

## Grow

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// make sure we evacuate the oldbucket corresponding
	// to the bucket we're about to use
	evacuate(t, h, bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}


func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	newbit := h.noldbuckets()
	if !evacuated(b) {
		// TODO: reuse overflow buckets instead of using new ones, if there
		// is no iterator using the old buckets.  (If !oldIterator.)

		// xy contains the x and y (low and high) evacuation destinations.
		var xy [2]evacDst
		x := &xy[0]
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.v = add(x.k, bucketCnt*uintptr(t.keysize))

		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
			y := &xy[1]
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.v = add(y.k, bucketCnt*uintptr(t.keysize))
		}

		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			v := add(k, bucketCnt*uintptr(t.keysize))
			for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
				top := b.tophash[i]
				if top == empty {
					b.tophash[i] = evacuatedEmpty
					continue
				}
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
				if t.indirectkey {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
				if !h.sameSizeGrow() {
					// Compute hash to make our evacuation decision (whether we need
					// to send this key/value to bucket x or bucket y).
					hash := t.key.alg.hash(k2, uintptr(h.hash0))
					if h.flags&iterator != 0 && !t.reflexivekey && !t.key.alg.equal(k2, k2) {
						// If key != key (NaNs), then the hash could be (and probably
						// will be) entirely different from the old hash. Moreover,
						// it isn't reproducible. Reproducibility is required in the
						// presence of iterators, as our evacuation decision must
						// match whatever decision the iterator made.
						// Fortunately, we have the freedom to send these keys either
						// way. Also, tophash is meaningless for these kinds of keys.
						// We let the low bit of tophash drive the evacuation decision.
						// We recompute a new random tophash for the next level so
						// these keys will get evenly distributed across all buckets
						// after multiple grows.
						useY = top & 1
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY {
					throw("bad evacuatedN")
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination

				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.v = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.indirectkey {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy value
				}
				if t.indirectvalue {
					*(*unsafe.Pointer)(dst.v) = *(*unsafe.Pointer)(v)
				} else {
					typedmemmove(t.elem, dst.v, v)
				}
				dst.i++
				// These updates might push these pointers past the end of the
				// key or value arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.v = add(dst.v, uintptr(t.valuesize))
			}
		}
		// Unlink the overflow buckets & clear key/value to help GC.
		if h.flags&oldIterator == 0 && t.bucket.kind&kindNoPointers == 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}
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

# FAQ

## 什么时候触发增量迁移动作？

insert或者delete操作时触发迁移动作的检查，相关的代码段如下：

```go
if h.growing() {
  growWork(t, h, bucket)
}
```



[深入理解 Go map：初始化和访问元素](<https://github.com/EDDYCJY/blog/blob/master/golang/pkg/2019-03-04-%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3Go-map-%E5%88%9D%E5%A7%8B%E5%8C%96%E5%92%8C%E8%AE%BF%E9%97%AE%E5%85%83%E7%B4%A0.md>)

[如何设计并实现一个线程安全的 Map ？(上篇)](<https://www.jianshu.com/p/cd41ca8741f4>)

[how-the-go-runtime-implements-maps-efficiently-without-generics](<https://dave.cheney.net/2018/05/29/how-the-go-runtime-implements-maps-efficiently-without-generics>)

