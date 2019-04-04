---
title: 使用Golang语言实现一个简单的Bitcask引擎的文件存储系统
date: 2016-04-25 23:45:22
tags:
- Bitcask
- 存储引擎
- Golang
categories:
- 存储引擎

description: 简化版的bitcask

---


`bitcask`是`bashro`的设计的一个底层存储引擎，主要应用于`Riak`产品中（`ps`:国内的`beansdb`的底层存储引擎也是使用`bitcask`，分布式上也是使用`dynamo`，并且他们也根据自己的实际应用做了相应的优化），其设计简单易懂，算法也是很简明的算法。其存储对象类型是`key/value`类型。

[riak bitcask github](https://github.com/basho/bitcask)

[go bitcask github](https://github.com/laohanlinux/bitcask)


<center>![](http://image.haha.mx/2013/04/03/middle/803545_68a109882550ec3556a2b19e277ddb10_1364992759.gif)</center>

<center>`talk is cheap, show me the code!` </center>


### 设计模型以及特点


- 所有的`key`都存储于内存中；所有的`value`都存储于磁盘中
- 以追加的方式写磁盘，即写操作是有序的，这样可以减少磁磁盘的寻道时间，是一种高吞吐量的写入方案，在更新数据时，也是把新数据追加到文件的后面，然后更新一下数据的文件指针映射即可
- 读取数据时，通过数据的指针以及偏移量即可，时间复杂度为`O(1)`，因为所有`key`都是存储于内存中，查找数据时，不用检索磁盘文件，这大大减少了检索时间
- `bitcask`有一个合并的时间窗口，当旧数据站到一定比例时，会触发合并操作，同时为了设计更简单，会把旧数据重新追加到可写文件中(`riak`里面的合并策略跟多，具体的合并策略可以去看它的源码)(`ps`:虽然自己实现了这个操作，但是还在测试阶段，应该有潜在的`Debug`,如果哪位有对这个项目有兴起，可以修复一下)

### 具体的组件设计图

`bitcask`的数据文件分为**只读文件**和唯一**一个读写文件**

<center>![图-1](http://laohanlinux.github.io/images/img/bitcask-1.png)</center>

为了加快索引的重建速度，每个数据文件对应一个`hint`文件，如：

<center>![图-2](http://laohanlinux.github.io/images/img/bitcask-2.png)</center>

`data`文件的格式如下：

```
crc32(4byte)|tStamp(4byte)|ksz(4byte)|valueSz(4byte)|key|value
```
这样通过`key`的大小和`value`的大小就可以找到`key`的位置和`value`的文件，但是如果`bitcask`重启后，直接扫描`data`文件来建立索引是一件非常耗时的工作，这时候`hint`文件就派上场了，`hint`文件格式如下：

```
tstamp(4byte)|ksz(4byte)|valuesz(4byte)|valuePos(8byte)|key
```
这样在可以跳过`value`的扫描，扫描速度自然就起来了，通过`valuePos`就可以直接找到文件的内容。

### 数据结构的设计

#### 文件映射的结构体

- `type BFile struct`

```go
// BFile 可写文件信息 1: datafile and hint file
type BFile struct {
	// fp is the writeable file
	fp          *os.File
	fileID      uint32
	writeOffset uint64
	// hintFp is the hint file
	hintFp *os.File
}
```
`fp`指向`Active data file`， `fileID`表示`Active data file`的文件名，`hintFp`表示`Active hint file`

- `type BFiles struct`

```go
// BFiles ...
type BFiles struct {
	bfs    map[uint32]*BFile
	rwLock *sync.RWMutex
}

```

`bfs`每一项表示一个文件索引项，直接使用`map`来存储不是一个高效的方法，以后再优化吧...

#### `key/value`结构体

- `type entry struct`

```go
type entry struct {
	fileID      uint32 // file id
	valueSz     uint32 // value size in data block
	valueOffset uint64 // value offset in data block
	timeStamp   uint32 // file access time spot
}
```

该结是`hint`文件的映射，`fileID`为`data`的文件名，`valueSz`表示值的大小，`valueOffset`表示`value`在`data`文件的索引位置，`timeStamp`表示`value`的存储时间（这个存储时间是会变的，因为在`merge`的时候，旧的数据会重新追加到`Active`文件中，这样这些旧的数据会重新洗牌，变成新的数据）.

- `type KeyDirs struct`

```go
// KeyDirs ...
type KeyDirs struct {
	entrys map[string]*entry
}
```

这个结构是主要的占内存的地方，因为所有`key`都存储于此，这个结构体由`hint`文件构建的.

这个结构体也是后续需要优化的地方，比如：`fileID`很多是相同的，可以将他们存储在一个数组中，`entry`只要存储数组的`fileID`索引即可。


- `type BitCask struct`

```go
// BitCask ...
type BitCask struct {
	Opts      *Options      // opts for bitcask
	oldFile   *BFiles       // hint file, data file
	lockFile  *os.File      // lock file with process
	keyDirs   *KeyDirs      // key/value hashMap, building with hint file
	dirFile   string        // bitcask storage  root dir
	writeFile *BFile        // writeable file
	rwLock    *sync.RWMutex // rwlocker for bitcask Get and put Operation
}
```

`bitcask`是最重要的结构体，是程序的入口，`oldFile`是只读文件的索引；`writeFile`是`Active file`的索引；`keyDirs`是`key`的索引。



### 关于Merge


为了节省空间，`bitcask`采用`merge`的方式剔除脏数据，`merge`期间会影响到服务的访问，`merge`是一件消耗`disk io`时间，用户应该错开`merge`的`io`高峰期.其中`merge`的触发也有很多种（触发不一定就会执行），如：

- 定时策略

用户自定义触发`merge`的时间
- 间隔策略
每隔一定的时间触发`merge`事件

其他等等.....

当`merge`时间触发时，`bitcask`就会根据用户自定的策略去决定是否执行`merge`操作，`merge`执行策略如：
- 定时策略
在用户定义的时间内执行`merge`操作，该操作会损耗服务的能力，用户应该避免高峰期，在低峰期时才执行该操作
- 容量策略
当胀数据达到一定的比例或者大小时，执行`merge`操作

其他的等等......


### 如何实现一个简单的`Merge`:

为简化设计，便于实现，`merge`操作把需要的`old file`文件重新扫描，如果记录是老的或者被删除了得，就过滤掉；需要保留的就按正常的操作重新插入到`active file`文件中。

<center>![](http://laohanlinux.github.io/images/img/bitcask-3.png)</center>

(`ps:本人只实现了一个简单的merge操作，由于比较忙，优化和策略方面还没全面`)

### TODO LIST 

- 优化hashmap
- 增加多种合并策略
- 减少锁的颗粒度

### 简单的操作

```go
package main

import (
        "os"

        "github.com/laohanlinux/bitcask"
        "github.com/laohanlinux/go-logger/logger"
)

func main() {
        os.RemoveAll("exampleBitcaskDir")
        bc, err := bitcask.Open("exampleBitcaskDir", nil)
        if err != nil {
                logger.Fatal(err)
        }
        defer bc.Close()
        k1 := []byte("xiaoMing")
        v1 := []byte("毕业于新东方推土机学院")

        k2 := []byte("zhanSan")
        v2 := []byte("毕业于新东方厨师学院")

        bc.Put(k1, v1)
        bc.Put(k2, v2)

        v1, _ = bc.Get(k1)
        v2, _ = bc.Get(k2)
        logger.Info(string(k1), string(v1))
        logger.Info(string(k2), string(v2))
        // override
        v2 = []byte("毕业于新东方美容美发学院")
        bc.Put(k2, v2)
        v2, _ = bc.Get(k2)
        logger.Info(string(k2), string(v2))

        bc.Del(k1)
        bc.Del(k2)
        logger.Info("毕业后的数据库：")
        v1, e := bc.Get(k1)
        if e != bitcask.ErrNotFound {
                logger.Info(string(k1), "shoud be:", bitcask.ErrNotFound)
        } else {
                logger.Info(string(k1), "已经毕业.")
        }
        v2, e = bc.Get(k2)
        if e != bitcask.ErrNotFound {
                logger.Info(string(k1), "shoud be:", bitcask.ErrNotFound)
        } else {
                logger.Info(string(k2), "已经毕业.")
        }

}

```
```go
> go run example/bitcask_main.go
2016/05/01 16:22:28 bitcask_main.go:28 [info] xiaoMing毕业于新东方推土机学院
2016/05/01 16:22:28 bitcask_main.go:29 [info] zhanSan毕业于新东方厨师学院
2016/05/01 16:22:28 bitcask_main.go:34 [info] zhanSan毕业于新东方美容美发学院
2016/05/01 16:22:28 bitcask_main.go:38 [info] 毕业后的数据库：
2016/05/01 16:22:28 bitcask_main.go:43 [info] xiaoMing已经毕业.
2016/05/01 16:22:28 bitcask_main.go:49 [info] zhanSan已经毕业.
```