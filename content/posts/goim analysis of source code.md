---
title: "Goim Analysis of Source Code"
date: 2016-12-04T15:03:15+08:00
draft: true
---

## Comet

`comet`为用户代理服务器，用于客户端的连接，根据情况可部署多个`comet`(部署机房选择以用户接入为基础，如：最近接入、按运营商接入)。

### 流程图

`comet`支持`tcp`以及`websocket`的方式和客户端交互，而和`logic`、`job`模块之间的交互采用`rpc`的方式，`comet`的主要流程图如下：

<center>

![websocket](http://laohanlinux.github.io/images/img/goim/comet-websocket.png)

</center>

<center> 图 1-1： comet websocket protoc</center>

<center>![tcp](http://laohanlinux.github.io/images/img/goim/comet-tcp.png)</center>

<center> 图 1-2： comet tcp protoc</center>

- 客户端首先连接到`comet`服务，`comet`调用`logic`来校验用户的合法性，`logic`会返回一个`subKey`给`comet`，该`subKey`成为该连接的唯一标示；
- 客户端接下来可以发心跳包给`comet`，同时，`job`服务将`MQ-Kafka`的消息转发到对应`comet`，`comet`再将其转发到对应的客户端

> 其中Logic前面可以加上一层4层代理服务器，如LVS。

### 主要逻辑代码分析

#### bucket

```go
type BucketOptions struct {
	ChannelSize   int 
	RoomSize      int   // 房间（Bucket.rooms）的初始化个数
	RoutineAmount int64 // Bucket.routines数组大小
    RoutineSize   int   // 房间信道通信(proto.BoardcastRoomArg)缓冲区的大小
}

// Bucket is a channel holder.
type Bucket struct {
	cLock    sync.RWMutex        // protect the channels for chs
	chs      map[string]*Channel // map sub key to a channel
	boptions BucketOptions
	// room
	rooms       map[int32]*Room // bucket room channels
	routines    []chan *proto.BoardcastRoomArg // 节点房间的总信道数
	routinesNum uint64 // 处理routines信道的goroutine个数，用于消费房播的信道消息
}
```

<center>![](http://laohanlinux.github.io/images/img/goim/bucket-goroutine.png)</center>

<center>图 1-3</center>

房间信息推送流程：

```shell
1、job通过rpc的方式发送msg到comet
2、comet内部将消息推送到至bucket.routines，bucket.routines以轮询的方式选出一个proto.BoardcastRoomArg信道进行消息转发
3、bucket.routines's worker 消费消息，根据msg'roomid从roomsMap(bucket.rooms)选出该room的所有客户端Channels，再将消息一一转发至client'Channel
4、dispatchWorker消费client'Channel消息，通过websocket或者tcp协议发送到客户端
```

单播推送流程：

单播相对于"房播（群播）"，简化了房间检索的步骤。

```shell
1、job通过rpc的方式发送msg到comet
2、comet内部根据消息的subKey，从buckets中找出对应的channel，将消息转发到对应的Channel
3、dispatchWorker消费client'Channel消息，通过websocket或者tcp协议发送到客户端
```

> 注：
>
> ```
> subKey的生成采用city32的hash算法，bucket是一个大小为n的hashMap slice，其主要目的是将数据切分成更小的块，从而降低资源的竞争。
> ```

多播以单播推送流程类似，广播也类似。

#### room

```go
type Room struct {
	id     int32        // 房间号
	rLock  sync.RWMutex // 锁
	next   *Channel     // 该房间的所有客户端的Channel
	drop   bool         // 标示房间是否存活
	Online int          // 房间的channel数量，即房间的在线用户的多少
}
```

`next *Channel`是一个双向链表，复杂度为`o(1)`，效率比较高。

#### tcp

协议：

<center>![](http://laohanlinux.github.io/images/img/goim/tcp-proto.png)</center>

<center>图 1-4</center>

- `BodySize`范围：`0<=BodySize<=MaxBodySize = int32(1<<10)`
- `PackSize`范围：`RawHeaderSize<=PackSize<=MacPackSize(RawHeaderSize+BodySize)`

[官网](https://github.com/Terry-Mao/goim/blob/master/doc/protocol.png)

#### websocket

[websocket协议](https://github.com/Terry-Mao/goim/blob/master/doc/proto.md)

#### RPC

`rpc`用于`logic`、`job service`模块的通信。

```go
// Push RPC
type PushRPC struct {
}
```

- func (this *PushRPC) Ping(arg *proto.NoArg, reply *proto.NoReply) error

心跳`rpc`

- func (this *PushRPC) PushMsg(arg *proto.PushMsgArg, reply *proto.NoReply) (err error)

根据指定的`subKey`推送消息

- func (this *PushRPC) MPushMsg(arg *proto.MPushMsgArg, reply *proto.MPushMsgReply) (err error)

根据指定的`subKey`一次推送多个消息，主要用于`群播`

- func (this *PushRPC) MPushMsgs(arg *proto.MPushMsgsArg, reply *proto.MPushMsgsReply) (err error)

多播

- func (this *PushRPC) Broadcast(arg *proto.BoardcastArg, reply *proto.NoReply) (err error)

广播

- func (this *PushRPC) BroadcastRoom(arg *proto.BoardcastRoomArg, reply *proto.NoReply) (err error)

按房间推送

- func (this *PushRPC) Rooms(arg *proto.NoArg, reply *proto.RoomsReply) (err error)

获取所有房间，包括下线的房间

#### ring

<center>![](http://laohanlinux.github.io/images/img/goim/comet-ring.png)</center>

<center>图 1-5</center>

`ring`是一个环形对象池，用于管理协议对象-`proto`，其内存结构如图所示，`rp`为可读游标，wp为可写游标，内存的大小为`4`的整数倍。

- func (r *Ring) Get() (proto *proto.Proto, err error)

获取一个`proto`对象的引用，用于读，不会移动可读游标

- func (r *Ring) GetAdv()

移动可读游标

- func (r *Ring) Set() (proto *proto.Proto, err error)

获取一个`proto`对象的引用，用于写，不会移动可写游标

- func (r *Ring) SetAdv() 

移动可写游标

- func (r *Ring) Reset()

重置`pool`对象

> 注：
>
> ```
> 如果wp移动过快，会影响rp游标指向的数据，如图：
> ```

<center>![](http://laohanlinux.github.io/images/img/goim/comet-ring-1.png)</center>

<center>图 1-6</center>

当程序运行到第`3`步的时候，`wp`已经又重新超过`rp`的索引了，这时候，第`6`个对象还没被`rp`读取过，但是它的数据已经被修改了，这样即使`rp`读取第6个对象，也是一个`dirty`对象。

#### Signal

信号处理模块，可用于在线加载配置，配置动态加载的信号为`SIGHUP`。

## Job

`Job`负责消费`kafka`消息，然后转发至`comet`。

### 单/多播

<center>![](http://laohanlinux.github.io/images/img/goim/Job=>多播.png)</center>

<center>图 2-1</center>

### 广播

<center>![](http://laohanlinux.github.io/images/img/goim/Job=> 广播.png)</center>

<center>图 2-2</center>

### 按房间推送

<center>![](http://laohanlinux.github.io/images/img/goim/Job=>按房间推送.png)</center>

<center>图 2-3</center>

### 主要逻辑代码分析

#### comet

<center>![](http://laohanlinux.github.io/images/img/goim/Job=>comet.png)</center>

<center>图 3-4</center>

```go
type CometOptions struct {
	RoutineSize int64 // 每个comet rpc goroutine个数
	RoutineChan int   // 每个协议通道的缓冲大小
}

type Comet struct {
	serverId             int32                      		// comet service id
	rpcClient            *xrpc.Clients              		// rpc连接对象
	pushRoutines         []chan *proto.MPushMsgArg  		// 单/多播信道
	broadcastRoutines    []chan *proto.BoardcastArg      	// 广播信道
	roomRoutines         []chan *proto.BoardcastRoomArg  	// 群播-房播信道
	pushRoutinesNum      int64 								// 单/多播协议信道=》用于循环
	roomRoutinesNum      int64 								// 房播-群播协议信道=》
	broadcastRoutinesNum int64 								// 广播协议信道=》
	options              CometOptions
}
```



- func (c *Comet) Push(arg *proto.MPushMsgArg) (err error)

循环选择一个`MPushMsgArg channel`，将消息推送至该信道

- func (c *Comet) BroadcastRoom(arg *proto.BoardcastRoomArg) (err error)

按照房间推送，循环选择一个`BoardcastRoomArg Channel`，将消息推送至该信道

- func (c *Comet) Broadcast(arg *proto.BoardcastArg) (err error)

广播，循环选择一个`BoardcastArg Channe`，将消息推送至该信道

- func (c *Comet) process(pushChan chan *proto.MPushMsgArg, roomChan chan *proto.BoardcastRoomArg, broadcastChan chan *proto.BoardcastArg)

`IO`复用协程，作为`pushChan`，`roomChan`，`broadcastChan`信道的消费者，消费完后，转发`comet service`

#### room

```go
type RoomBucket struct {
	roomNum int 
	rooms   map[int32]*Room
	rwLock  sync.RWMutex
	options RoomOptions
	round   *Round
}
type RoomOptions struct {
	BatchNum   int  			//汇总阈值
	SignalTime time.Duration
}
```

`room`模块用于接收`kafka`的`push`模块发送的消息，每个`room`都有一个协程，其协程的信道缓冲区的大小为`RoomOptions.BatchNum`的两倍。

> 注： 
>
> ```
> 1、room协程在接收到的消息条数>=BatchNum*2或者timeout时，才会触发转发消息的行为(转发至broadcastRoutines)，即其具有汇总操作。
> 
> 2、房间的消息使用了Libs/bytes.Writer进行汇总缓存。
> ```

#### push/kafka

`push/kafka`模块用于预处理消息，消息从`kafka集群`流出，经过`kafka`模块转发至`push`模块，`push`模块对消息预处理/过滤/分类，然后发至不同的`comet 信道`中，具体使用请参照`Job的`前3章图.

消息的分类：

- `KAFKA_MESSAGE_MULTI`=>多播

- `KAFKA_MESSAGE_BROADCAST`=>广播

- `KAFKA_MESSAGE_BROADCAST_ROOM`=>群播/房播

#### Round/RoundOptions

时钟管理器



## Logic

<center>![](http://laohanlinux.github.io/images/img/goim/logic-router.png)</center>

<center>图 3-5</center>

`Logic`是`goim`是主要业务处理模块，负责的内容有：

- 注册/注销
- 验证
- 消息`Push`代理

### 协议

#### Push协议

[linked](https://github.com/Terry-Mao/goim/blob/master/doc/push.md)

其中`ensure`参数是额外的参数，用于控制消息是否必达，为布偶值。

#### 其他http协议

##### 删除comet service

| 接口名                                | URL             | 访问方式 |
| ------------------------------------- | --------------- | -------- |
| 删除`comet service`                   | `/1/server/del` | `POST`   |
| 获取所有`comet service`或者`room`信息 | `/1/count`      | `POST`   |

请求例子：

- `/1/server/del`

```
curl -XPOST "http://127.0.0.1:7172/1/server/del?server=1
```

- `/1/count`

```
curl -XPOST "http://127.0.0.1:7172/1/count?type=room
```

### 主要逻辑代码分析

#### Auth

```go
type Auther interface {
	Auth(token string) (userId int64, roomId int32)
}
```

用户验证模块

#### router

负责与`Router Service`交互，多个`Router Service`采用的是一致性`hash`算法，`hash`的输入为`Router Serviceid`，默认所有的`Router Service`权值是一样的，如果需要控制不同的权值，可以在配置`router service`的时候加多个端口实例或者同一个节点配置成多个`serviceid`标签。

> 注：
>
> ```
> 一致性hash没有实现动态扩容，即没有自动平衡，所以不能够动态改变Router service映射配置，且每个Logic节点的Router Service配置项需保持一致。
> ```

- func getRouterByServer(server string) (*xrpc.Clients, error)

通过`router serverid`获取对应的`router service client`

- func getRouterByUID(userID int64) (*xrpc.Clients, error)

通过`userID`获取`router service client`

- func getRouterNode(userID int64) string

通过`userID`获取到`Router 节点`

- func connect(userID int64, server, roomId int32) (seq int32, err error)

注册、验证用户，返回一个自增序列号(`cometServiceID_incr`)

- func disconnect(userID int64, seq, roomId int32) (has bool, err error)

注销一个用户

- func delServer(server int32) (err error)

下线一个`comet service`

- func allRoomCount(client *xrpc.Clients) (counter map[int32]int32, err error

获取所有房间个数

- func allServerCount(client *xrpc.Clients) (counter map[int32]int32, err error)

获取所有`comet-service`

- func genSubKey(userId int64) (res map[int32][]string)

通过用户`ID`生成`subKey`；同一个用户可以同时多处登陆或者同处多实例登陆，它们都会被同等对待。

`subKey生成代码：`

```go
func encode(userId int64, seq int32) string {
	return fmt.Sprintf("%d_%d", userId, seq)
}
```

`seq`为每个`comet service`的自增长序索引，假设`comet service 1`接受到一个用户请求，用户的`ID`为`10`，而此时`Router buckets`的`seqIdx`为`9`，则`subKey = 10_9`

- func getSubKeys(res chan *proto.MGetReply, serverId string, userIds []int64) | func genSubKeys(userIds []int64) (divide map[int32][]string)

并行获取多个用户信息，返回值为`map[comet.serverId][]subkey`.

#### rpc

```go
// RPC
type RPC struct {
	auther Auther
}
```

`LogicService`用于心跳的检测以及客户端的注册/注销。

- func (r *RPC) Connect(arg *proto.ConnArg, reply *proto.ConnReply) (err error)

注册

- func (r *RPC) Disconnect(arg *proto.DisconnArg, reply *proto.DisconnReply) (err error)

注销

> 注：
>
> ```
> goim不支持离线消息，如果需要支持离线消息的推送，应hook该模块，实现离线消息推送逻辑
> ```

## Router

### 主要逻辑代码分析

#### session

```go
type Session struct {
	seq     int32                     // 序列号自增标记器
	servers map[int32]int32           // seq:server
	rooms   map[int32]map[int32]int32 // roomid:seq:server with specified room id
}
```

客户端会话信息管理，以用户`id`为单位，即每个用户有且拥有一个`session`，`session`包含了用户每个连接的`comet service`信息，以及每个连接所属的`roomid`。

- func (s *Session) Put(server int32) (seq int32)

关联一个`subKey`和`session`

- func (s *Session) PutRoom(server int32, roomId int32) (seq int32)

关联一个`subKey`到`comet servicey` 以及`room`

- func (s *Session) Servers() (seqs []int32, servers []int32)

返回`session`关联的所有`comet service`信息

- func (s *Session) Del(seq int32) (has, empty bool, server int32)

删除指定的`subKey`所关联的`Session.Servers`

- func (s *Session) DelRoom(seq int32, roomId int32) (has, empty bool, server int32)

删除指定的`subKey`、`roomid`所关联的`Session.rooms`

- func (s *Session) Count() int

返回`session`所关联的所有`comet service`信息

#### bucket

```go
type Bucket struct {
	bLock             sync.RWMutex
	server            int                       // session server map init num
	session           int                       // bucket session init num
	sessions          map[int64]*Session        // userid->sessions, 一个可能同时多处登陆
	roomCounter       map[int32]int32           // roomid->count
	serverCounter     map[int32]int32           // server->count
	userServerCounter map[int32]map[int64]int32 // serverid->userid count
	cleaner           *Cleaner                  // bucket map cleaner
}
```

- func (b *Bucket) counter(userId int64, server int32, roomId int32, incr bool)

增加一个用户或者减少一个用户

- func (b *Bucket) Put(userId int64, server int32, roomId int32) (seq int32)

关联`channel(session)`到指定的用户`id`，`comet id`以及`room id`

- func (b *Bucket) Get(userId int64) (seqs []int32, servers []int32)

指定用户的`session`信息

- func (b *Bucket) GetAll() (userIds []int64, seqs [][]int32, servers [][]int32)

返回该`bucket`的所有用户、`subKey`、`comet`信息

其他函数省略...

#### cleaner

```go
type CleanData struct {
	Key        int64
	expireTime time.Time
	next, prev *CleanData
}

type Cleaner struct {
	cLock sync.Mutex
	size  int
	root  CleanData
	maps  map[int64]*CleanData
}
```

`lru`对象管理器，只负责管理，不负责触发`GC`，`GC`交给`Runtime`处理。

主要应用于客户端的`session`管理，定时处理掉一些过期的`session`对象。

> 1、数据结构使用map和双向列表，map用于快速检索；
>
> 2、双向链表用于快速剔除数据：因为从map中剔除数据，map的结构会实时改变，每剔除一个都得再次从起点开始遍历map，而使用链表不用重新遍历，时间复杂度为`O(logN)`

## Libs

### 缓冲io-bufio

#### Reader

```go
type Reader struct {
  	buf []byte
  	rd	io.Reader
  	r, w int
  	err error
}
```

`Reader`是一个具有缓存的可读`IO`。

<center>![](http://laohanlinux.github.io/images/img/goim/bufio-reader.png)</center>

<center>图 5-1</center>

主要函数

- func (b *Reader) Reset(r io.Reader) 

重置`IO`，可读游标重置为`0`

- func (b *Reader) ResetBuffer(r io.Reader, buf []byte)

重置`IO`，可读游标重置为`0`，且`b.buf `变为`buf`

- func (b *Reader) Peek(n int) ([]byte, error)

窥探缓存的`n`个字节，可读游标维持不变，可写游标可能会改变; 

如果可窥探的数据少于`n`，则调用`b.fill()`尝试读取远端数据用于填充`b.buf`.

- func (b *Reader) Pop(n int) ([]byte, error)

返回`[b.r:b.r+n]`处的数据，该函数会调用`b.Peek`，而且会改变可读游标和科协游标

- func (b *Reader) Discard(n int) (discarded int, err error)

丢弃`n`个数据；如果缓冲区的可读数据小于`n`，则一致调用`b.fill()`尝试读取远程的数据，直到`b.buf`的可读缓冲区大于或者等于`n`或者出现网络异常为止

- func (b *Reader) Read(p []byte) (n int, err error)

读取缓冲区数据；

如果缓冲区为空，为了提高效率，避免应用层的数据拷贝（`kernel net stack=>b.buf==>p`），直接将`kernel net stack`拷贝到`p []byte`，并且调用`f.fill()`整理缓冲区.

- func (b *Reader) fill()

<center>![](http://laohanlinux.github.io/images/img/goim/bufio-reader-fill.png)</center>

<center>图 5-2</center>

`fill`读取远程新块数据到本地缓冲区的`b.buf`

`fill`会有数据的移动，如`图 5-2`所示。

#### Writer

```go
type Writer struct {
	err error
	buf []byte
	n   int
	wr  io.Writer
}
```

`Writer`是一个具有缓存的可写`IO`.

- func (b *Writer) Reset(w io.Writer) {

重置`IO`，可写游标重置为`0`，句柄变为`w`

- func (b *Writer) ResetBuffer(w io.Writer, buf []byte)

重置`IO`，可写游标重置为`0`，句柄变为`w`，缓冲区变为`buf`

- func (b *Writer) flush() error

刷新缓冲区，将本地缓冲区的数据发送至内核网络栈；

游标被重置（不一定会被重置为0，可能为其他值，因为本地缓冲的数据大于内核可写缓冲区，这时还会造成数据的搬迁）。

<center>![](http://laohanlinux.github.io/images/img/goim/bufio-writer.png)</center>

- func (b *Writer) Available() int

可写字节数

- func (b *Writer) Buffered() 

已写缓冲大小

- func (b *Writer) Write(p []byte) (nn int, err error)

将`p []byte`写到缓冲区或者直接写到网络内核栈（此时缓冲区已满），可能造成可写游标的移动

- func (b *Writer) WriteRaw(p []byte) (nn int, err error)

和b.Write()类似

- func (b *Writer) Peek(n int) ([]byte, error)

窥探可写缓冲区剩余值，如果可写缓冲区的剩余值小于`n`，会调用`flush`，可写游标相应可能会改变

#### timer

`timer`是一个最小堆算法实现的时钟对象，串行执行时钟任务，所以时钟任务应该尽量小，`timer`不适合太耗时的任务，当然用户可以控制时钟任务的并发。

### bytes

#### Pool

```go
type Buffer struct {
	buf  []byte
	next *Buffer // next free buffer
}
type Pool struct {
	lock sync.Mutex
	free *Buffer
	max  int
	num  int
	size int
}
```

`pool`内存组织如下，`pool`是一个链式存储的`栈`，数据从栈顶出，同时数据也从栈顶回收。

<center>![](http://laohanlinux.github.io/images/img/goim/bytes.png)</center>



- func (p *Pool) Get() (b *Buffer)

获取一个缓冲区

- func (p *Pool) Put(b *Buffer)

归还一个缓冲区

- func (p *Pool) grow()

重置`Pool`对象

> 注：
>
> ```
> pool 是一个不限大小的内存池，如果栈没有数据了，那么pool会调用glow()重新生成数据，所以最后可能造成的内存架构如下图所示
> ```

<center>![](http://laohanlinux.github.io/images/img/goim/bytes-2.png)</center>

如果租借的速度大于归还的速度，会造成内存的溢出。

#### Writer

```go
type Writer struct {
	n   int 	//游标
	buf []byte
}
```

`writer`是一个具有缓冲的可写`IO`。

- func (w *Writer) Size()

缓冲区的大小

- func (w *Writer) Reset()

重置缓冲区，游标重置为`0`

- func (w *Writer) Buffer() []byte

返回缓冲区的内容

- func (w *Writer) Peek(n int) []byte

窥探缓冲区的`n`个字节，如果缓冲区的剩余空间小于`n`，则会调用`w.grow()`自增长数据缓冲区，游标会移动

- func (w *Writer) grow(n int)

按照`2`倍的大小增长缓冲区，会发生数据的移动

### net

#### xrpc

根据原生的`rpc`封装的，其调用采用异步的方式，具有重连功能。

`xrpc`并没有做负载均衡的工作，只是简单做一下容灾而已，相对来时不是很灵活，用户可以稍微修改一下，就支持负载均衡了。

同时`xrpc`也没有在代码层面上实现`[host:por]`连接池，即同一个`[host:port]`配置只会有一个`rpc`长链接，除非增加多配置。

#### proto

主要的`rpc`协议说明：

##### job

```go
type KafkaMsg struct {
	OP       string   `json:"op"` 				//操作类型
	RoomId   int32    `json:"roomid,omitempty"` //房间号
	ServerId int32    `json:"server,omitempty"`	//comet id
	SubKeys  []string `json:"subkeys,omitempty"`
	Msg      []byte   `json:"msg"`
	Ensure   bool     `json:"ensure,omitempty"`	//是否强推送(伪强推)
}
```

##### logic

客户端上线：

```go
// 用于comet发送客户端的校验信息
type ConnArg struct {
	Token  string // Token
	Server int32  // comet id
}

// logic 校验应答
type ConnReply struct {
	Key    string 	// subKey
	RoomId int32	// 房间号
}
```

客户端下线：

```go
// 用于comet发送客户端连接下线
type DisconnArg struct {
	Key    string	// subKey
	RoomId int32	// 房间号
}
// 应答客户端下线
type DisconnReply struct {
	Has bool
}
```

##### comet

`Push RPC`模块

心跳：

```go
type NoArg struct {
}

type NoReply struct {
}
```

`Job---->comet`

```go
// 单播
type PushMsgArg struct {
	Key string	//subKey
	P   Proto
}
type NoReply struct {
}

// 把某条消息推送给多个subKey
type MPushMsgArg struct {
	Keys []string
	P    Proto
}

type MPushMsgReply struct {
	Index int32
}
```

// 多播
type MPushMsgsArg struct {
	PMArgs []*PushMsgArg
}

type MPushMsgsReply struct {
	Index int32
}

// 广播
type BoardcastArg struct {
	P Proto
}

// 吧某条消息推送给某个房间的所有channels
type BoardcastRoomArg struct {
	RoomId int32
	P      Proto
}

type RoomsReply struct {
	RoomIds map[int32]struct{}
}

```
##### router

增加用户：

​```go
type PutArg struct {
	UserId int64
	Server int32
	RoomId int32
}
type PutReply struct {
	Seq int32	// 序列号
}
```

移除用户：

```go
type DelArg struct {
	UserId int64
	Seq    int32
	RoomId int32
}

type DelReply struct {
	Has bool	//	是否存在目标用户
}
```

其他：

```go
// 剔除comet server
type DelServerArg struct {
	Server int32
}

// 获取用户信息
type GetArg struct {
	UserId int64
}

// 获取Router的所有信息
type GetReply struct {
	Seqs    []int32
	Servers []int32
}

type GetAllReply struct {
	UserIds  []int64
	Sessions []*GetReply
}

// 获取多个用户信息
type MGetArg struct {
	UserIds []int64
}

type MGetReply struct {
	UserIds  []int64
	Sessions []*GetReply
}

// 返回所有连接个数
type CountReply struct {
	Count int32
}

// 获取特定房间的所有连接
type RoomCountArg struct {
	RoomId int32
}

type RoomCountReply struct {
	Count int32
}

// 获取所有房间的连接个数
type AllRoomCountReply struct {
	Counter map[int32]int32
}

// 获取所有的comet server个数
type AllServerCountReply struct {
	Counter map[int32]int32
}

// 获取所有的用户个数
type UserCountArg struct {
	UserId int64
}

type UserCountReply struct {
	Count int32
}
```

##### 推送协议

参照官网