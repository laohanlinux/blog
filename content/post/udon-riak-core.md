---
title: udon_riak_core
date: 2016-03-13 22:10:52
tags:
- riak
- riak core
categories: 
- Distribute System
description: "使用riak core 来写分布式存储系统."
---


最近要做文件分片传输的接口，起初是想分片容灾的，有不想借助第三方存储中间状态，有可以自动扩展接口服务能力，是试下一下，`riak core`

刚好可以满足这些需求，`dynamo`果然是个好东西。 好吧，最后还是没采用这个架构做，至于原因嘛。。。。。。。。。

不过也重新认识了`riak core`的实现方式。下面参照`udon`这个项目，做一个小总结。

github: https://github.com/mrallen1/udon

udon: a distributed static file web server


至于riak core 不太熟的同学，可以看一下博客前面关于riak core 的介绍，那些介绍我也是从网上收集回来的，特别是翻译那基本，有很多问题，

建议大家看英文版的。

这里主要介绍一下`udon`的`udon_vnode.erl`模块， 这也是`riak core` 的主要回调模块。

主要模块函数：

- handle_command
`handle_command` 是vnode的回调模块，会根据用户的命令来做相应的回调，如果只是把`riak core`作为计算节点来用，那么其实实现这个函数就

已经可以满足我们的要求了，主要的业务处理也是这里。

`udon` 实现了3个命令的回调-`ping`, `store`, `fetch`， 其中主要是`store` 和`fetch`。 `store`实现存储的业务，根据key/value来存储一个文件，
key可以看做文件路径，value就是文件的内容，把他们以hash的值存储为`hash`和`hash.meta`文件， 并且文件的内容还有版本控制；`fetch`会根据key来获取value。

- handle_handoff_command
`handle_handoff_command`也是一个主要的模块，这个模块会在节点发生异常时被调用，如节点的`down 机`，增加节点，删除节点，也就是说集群由于某种原因，
本属于这个节点的数据，现在又不属于它了，那么这函数会被调用，这个函数会把这些数据搬到相应的节点中去。
`handle_handoff_command`主要是把这个vnode的全部数据检索出来，然后交给下一步处理。

- handoff_starting(TargetNode, State)
如果返回`{true, State}`， 则表示开始搬运vnode的内容到TargetNode 中去。

- handoff_cancelled(State)
如果传输过程中发生出错，这个函数也会被调用；可以使用这个函数撤销`handoff_starting`的任务。

- handoff_finished
当所有的数据都被传输到TargetNode后，这个函数会被调用。

- encode_handoff_item(K,V)
对要差传输的数据进行序列化操作， `udon`把V的内容进行了`encode`

- handle_handoff_data()
这个和`encode_handoff_item`相反，是一个反序列化操作，其实这个函数就是`TargetNode`接到数据的时候调用的函数，
`udon`又把街道数据存储到`TargetNode`中去了。

- is_empty(State)
如果返回{false, State} 表示该vnode没有数据传输，如果是{true, State}则表示有数据要传输。

- delete(State) ->
如果已经传递完数据了，那么该函数将会被调用，主要写一些清理的工作。
    {ok, State}.

- handle_coverage
迭代输出所有的数据， `udon`没有实现这个功能，可以参照`riak kv`的实现方式来实现该功能。

- handle_exit

*详细的内容，建议看前面的riak core的详细教程*