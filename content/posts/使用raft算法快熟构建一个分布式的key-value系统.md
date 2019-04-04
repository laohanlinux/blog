---
title: 使用raft算法快速构建一个分布式kv系统
date: 2016-04-25 23:44:25
tags:
- Golang
- raft 
- 分布式存储系统
- 分布式
categories: 
- Distribute System
description: 一个基于hashicorp raft算法的分布式key/value系统. 
---


`raft`是一种类似于`paoxs`的分布式算法，相对于`paxos`算法，`raft`更容易于理解以及实现，这也是一种典型的`半数协议算法`。这里不详细介绍`raft`算法，有兴趣的同学可以参照一下下面的文章：

[raft Algorithm](https://laohanlinux.github.io/2016/03/13/raft%E7%AE%97%E6%B3%95/)

本次教程实现的`key/value`分布式存储系统的`github`地址为：[riot github](https://github.com/laohanlinux/riot)

## 如何快速使用`hashicorp raft`搭建一个简单的分布式系统

`raft`算法种类的实现有很多，比较流行的是`etcd raft`和`hashicorp raft`，这两个都是`Go`语言实现的`raft`算法库，并且都大量应用到生产环境中，可靠性高。由于本人对`hashiro raft`熟悉一点，刚好又对`Go`语言有兴趣，所以选择了`hashicorp raft`来实现一个简单的分布式`key/value`系统.

系统设计的基本目标：

- 具有容错性
- 可以在线自动线性扩展节点
- 可以自动增删节点
- 具体增删查改操作
- 可以适配多种存储引擎
- 支持日志/快照重建

`raft`的请求处理流程：


<center>![](http://laohan.qiniudn.com/raft/raft_1.png)</center>

除此之外还有一个状态机快照的模块。

使用`hashicorp raft`构建一个分布式存储系统时，主要需要实现的模块有：

- FSM

`fsm`为`raft`的日志状态机

- Snapshot

`Snapshot`为`raft`的快照对象

这两个对象是比较重要的，基本上所有的核心都基于这个两个对象进行构建。

## `Riot`的主要组件

- 模块图

<center>![](http://laohan.qiniudn.com/raft/riot_2.png)</center>

- 流程图

<center>![](http://7rflb0.com1.z0.glb.clouddn.com/raft/riot_4.png)</center>
### Backend Store

```go
type RiotStorage interface {
	Get([]byte) ([]byte, error)
	Set([]byte, []byte) error
	Del([]byte) error
	Rec() <-chan store.Iterm
}
```
`Rec()`返回一个只读的信道，该信道用于`riot`的快照系统。


为了便于扩展更多的存储后端，该接口对外开放，有兴趣的朋友只要实现这个接口即可。

```go
type leveldbStorage struct {
	*leveldb.DB
	c chan Iterm
	l *sync.Mutex
}
```

`Riot`目前采用的存储后端为`leveldb`，未了会增加`Bitcask`[github-link](https://github.com/laohanlinux/bitcask)存储后端,

```go
type Iterm struct {
	Err   error
	Key   []byte
	Value []byte
}
```

其中当`iterm.Err = ErrFinished`是表示所有的`key`已迭代完毕。


为了确保一个节点中，同一时间只有一个快照操作，需要操作（`raft`本身就也确保了，为了更加保险，建议还是加上锁，毕竟`IO`才是这个系统的瓶颈，锁的损耗可以忽略）。

实现起来还是比较明了的。

### FSM、SnapShot

`raft fsm`:

```go
type FSM interface {
    
    Apply(*Log) interface{}
  
    Snapshot() (FSMSnapshot, error)
    
    Restore(io.ReadCloser) error
}
```
用户自定义的`fsm`只要实现这个接口即可。

`riot fsm`:
```go
type StorageFSM struct {
	l  *sync.Mutex //互斥锁
	rs RiotStorage //存储后端
}
```

这里的互斥锁是必须的，因为`fsm`的`Apply`和`Snapshot`不是线程安全的。

其中业务操作都会应用到`Apply`方法中，所以把所有的业务请求都按一定的格式打包即可，然后再解包，根据包的`action`类型，做相应的操作。其中操作主要有：

- SET
- DEL 
- SHARE

> 注:
>
>1、`SHARE` 操作类型用于Riot节点通信有的，`Riot`集群在启动的时候，他们之间的有些信息需要同步，目前用于同步Leader的RPC地址和端口
>
>2、为了性能，`GET`操作并没有放在Apply方法中，所以GET请求会有404的情况出现，未来会把查询请求是否经过Leader的权限交给用户.
>
>3、`DEL`和`SHARE`操作全部交给Leader处理,再由Leader下发到Follower节点

`Shapshot`方法会在快照时被执行,这时候,只要把所有的`key/value`传递给`FSMSnapshot`对象即可.在`Riot`中.只要传递`RiotStorage`对象给`StorageSnapshot`即可;

然后`StorageSnapshot.Persist`方法将被调用,`StorageSnapshot`只要遍历这些数据,按一定的格式快照到SnapshotSink中即可.

<center> ![](http://7rflb0.com1.z0.glb.clouddn.com/raft/Riot_3.png)</center>
当服务重启时,`Riot`会检测是否存在快照,如果存在快照,快照的数据就会被`StorageFSM.Restore`进行重建;重建完后,`raft`会根据日志的索引,重放那些没有被快照到日志条目,这样
所有的数据就恢复完成了.


### Cluster

```go
type Cluster struct {
	Dir         string
	R           *raft.Raft
	Stores      *raft.InmemStore
	FSM         *fsm.StorageFSM
	Snap        raft.SnapshotStore
	Tran        raft.Transport
	PeerStorage raft.PeerStore
}
```
好吧，偷个懒，`raft.Raft`访问权限直接暴露出......

- PeerStorge

节点地址列表的存储对象
- Tran 

节点网络通信对象
- Snap

日志快照
- FSM

状态机
- Stores

日志存储对象


### RPC

节点之间的业务通信主要采用`gRPC`方式.


- pb


```probuffer
syntax = "proto3";

package pb;

service RiotGossip {
    rpc OpRPC(OpRequest) returns (OpReply) {}
}

message OpRequest {
    string op = 1;
    string key= 2;
    bytes value = 3;
}

message OpReply {
    int32 status = 1;
    string msg = 2;
    int32 errCode = 3;
}
```
`OpRequest.op`表示操作类型，其取值为：

```go
CmdGet   = "GET"
CmdSet   = "SET"
CmdDel   = "DEL"
CmdShare = "SHARE"
```
总共4做操作类型，其中`SHARE`和`CmdGet`这两类型不会影响到日志快照。

(`PS`：`OpReply`结构需要调整一下，如果`GET`操作增加一致性，起码需要增加一项`value`)

- RiotRPCClient

```go
type RiotRPCClient struct {
	l    *sync.RWMutex
	conn map[string]*grpc.ClientConn
}
```
每个服务器到客户端的连接有且只有一个`conn`.

- RiotRPCService

```go
type RiotRPCService struct{}
```

`RiotRPCService`只要实现`OpRPC`这个方法即可

### Http Interface

所有的业务入口都是`http`请求，包括集群管理

- RiotHandler

```go
type RiotHandler struct{}
```
`RiotHandler`更具请求类型来判定操作类型.

- AdminHandlerFunc

`AdminHandlerFunc`可以获取到集群的`Leader`地址、集群节点地址信息、节点角色信息、节点的`rpc`地址信息；还有删除集群中的某一个节点以及把新节点加入到集群中。

## Example

脚本位于'riot/tool'目录下：

### 启动集群

```erlang
bash-3.2$ ./cluster1.sh
raft: {127.0.0.1 12000 [127.0.0.1:12000] raft0/raft_peer_storage raft0/raft_snapshot_storage raft0/storage_backend_path raft0/raft_log_path raft0/apply_log_path true}
rpc: {127.0.0.1 32123}
leader rpc: { }
server:{localhost 8080}
raft: {127.0.0.1 12001 [127.0.0.1:12001] raft1/raft_peer_storage raft1/raft_snapshot_storage raft1/storage_backend_path raft1/raft_log_path raft1/apply_log_path false}
rpc: {127.0.0.1 32124}
leader rpc: { }
server:{localhost 8081}
2016/05/02 23:35:11 admin_handler.go:124 [info] The Leader Name is :127.0.0.1:12000
2016/05/02 23:35:11 admin_handler.go:130 [debug] 127.0.0.1:12001will join the cluster, leader is :127.0.0.1:12000
2016/05/02 23:35:11 [DEBUG] raft-net: 127.0.0.1:12001 accepted connection from: 127.0.0.1:61792
2016/05/02 23:35:11 riot.go:147 [error] 127.0.0.1:12000timed out enqueuing operation
2016/05/02 23:35:11 riot.go:124 [info] {<nil> 0 0.0030684640000000003 {0 0 <nil>}}
2016/05/02 23:35:11 [DEBUG] raft-net: 127.0.0.1:12001 accepted connection from: 127.0.0.1:61793
raft: {127.0.0.1 12002 [127.0.0.1:12002] raft2/raft_peer_storage raft2/raft_snapshot_storage raft2/storage_backend_path raft2/raft_log_path raft2/apply_log_path false}
rpc: {127.0.0.1 32125}
leader rpc: { }
server:{localhost 8082}
2016/05/02 23:35:13 admin_handler.go:124 [info] The Leader Name is :127.0.0.1:12000
2016/05/02 23:35:13 admin_handler.go:130 [debug] 127.0.0.1:12002will join the cluster, leader is :127.0.0.1:12000
2016/05/02 23:35:13 [DEBUG] raft-net: 127.0.0.1:12002 accepted connection from: 127.0.0.1:61797
2016/05/02 23:35:13 riot.go:124 [info] {<nil> 0 0.004264005 {0 0 <nil>}}
2016/05/02 23:35:13 [DEBUG] raft-net: 127.0.0.1:12002 accepted connection from: 127.0.0.1:61798
```

### 查看集群信息

```erlang
curl  "http://localhost:8080/admin/status"
{
  "results": "Leader",
  "error": 0,
  "time": 3.9520000000000004e-06
}
```

```erlang
curl  "http://localhost:8080/admin/peer"
{
  "results": [
    "127.0.0.1:12002",
    "127.0.0.1:12000",
    "127.0.0.1:12001"
  ],
  "error": 0,
  "time": 0.017415839000000002
}
```

```erlang
curl  "http://localhost:8080/admin/status"
{
  "results": "Leader",
  "error": 0,
  "time": 2.7490000000000003e-06
}
```

### 基本操作

- SET

```erlang
curl http://localhost:8080/riot\?key\=a -d '1024'
```

- GET

```erlang
curl http://localhost:8081/riot\?key\=a
1024%
```

- DEL

```erlang
curl -XDELETE http://localhost:8082/riot\?key\=a
```

- GET

```erlang
curl  http://localhost:8082/riot\?key\=a
{"errCode":40004,"msg":"not found"}%
```
## TODO

- 增加快照的压缩算法,提高压缩效率
- 增加跟多的监控信息
- 增加多种后端存储引擎的支持
- 优化代码结构
- 重新设计`http api`，新的`api`为`REST`风格