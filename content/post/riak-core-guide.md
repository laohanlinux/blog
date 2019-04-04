---
title: riak core 导读 
date: 2016-03-13 22:13:13
tags:
- riak
- riak core
categories: 
- Distribute System
description: "riak core 原理分析的导读部分，后续会发布更加细节的部分"
---

这一部分的文章整理主要参照于 `try-try-try` 和 *书写是对思维的缓存 / 好记性不如烂博客* `http://cryolite.iteye.com/`

# Riak Core Guide

`Riak Core` 并不会涉及到数据的物理存储， `demo`参照 **【try try try】**的`github`。

在`3try`的例子中，可以看到，`mfmn`是业务逻辑`application`， 而`riak_core application`为这个`mfmn`系统提供了分布式的系统架构支持。

`mfmn_vnode_master` 是`riak_core_vnode_master`模块运行时的一个进程注册名，这个进程为容纳业务模块代码的容器运行。在进程启动时系统的业务逻辑会嵌到这个进程中。


## 3Try RTS Demo

- 服务及其API

接收日志数据的服务在`rts_entry_vnode`模块中实现,统计日志的服务在`rts_stat_vnode`模块中实现。
同时,这两类服务对外提供的调用API也在对应的模块中。
每当接收日志服务收到一条日志,会用正则表达式分析日志,根据分析结果调用统计服务对日志信息进行
计数。
最后由一个`rts`模块作为`Facade`包装这两个服务,统一对外提供API。

BTW:由于每个日志信息的上传都要新建一个HTTP连接,这里就成了系统输入的性能瓶颈。因此即使使用 多个物理节点时也不会感受到系统性能的提高。


- 实现

这里的重点是了解和学习`Riak Core`的`vnode`接口如何使用实现业务逻辑:基于`riak_core_vnode behaviour`实现相关回调函数。
对于每个`partition`,会有一个`rts_stat_vnode`进程负责该分区内日志数据的各种状态统计,这个`vnode`进程内维护一个字典数据结构(`dict`),
用来存储这个`partition`上的各种日志状态。字典的`key`是状态名, `value`是整数或者`list`。
不过我觉得这个`rts`例子的处理逻辑似乎有问题,不能处理多个用户的录入。
`riak_core_vnode behaviour`的回调函数:

生命周期回调函数:

`init(Partition)` 初始化`vnode`进程的状态(类似`gen_server/gen_fsm`的状态),回调函数的参数是代表此 `vnode`负责的分区的`ring`整数;

`terminate(Reason, State)`,`handle_exit/3` 当与`vnode`进程有`link`的其它进程崩溃时被调用.

用户定制的`riak_core_vnode`回调函数模块被称为`vnode_moudles`,可以调用 `application:get_env(riak_core, vnode_modules)`. 查询当前应用的`vnode`模块。这些在 `riak_core:register_vnode_module`注册。

## Riak Core 基本原理

`Riak Core`是一个分布式系统的开发框架,如果你的应用要采用分布式架构,凑巧你又选中了分布 式哈希表这种分布式架构,则可以考虑使用`Riak Core`,不然自己从底层重新实现一个分布式哈希 表实在是太麻烦了。

- 1.基于`dynamo`设计的`Riak Core`

通过某种`hash`算法,对数据的某一特征计算哈希值,每份数据会对应着一个唯一的整数。这样处 理后,这些数据将均匀的映射到一个整数区间上。
对于`Riak Core`,它管理着一个整数范围为`[0-2^160]`整数空间,这个空间形成一个首尾相连的环。
`Riak Core`把这个环平均划分成多个分区`partition`(默认是64个分区,当前的版本不能动态修改这 个参数,据说将来会),`partition`在环上的`Token`作为此`partition`的唯一标识`ID`,每个`partition`交给一个或多个不同类型的`vnode`进程负责,每类`vnode`提供一套服务功能。`partition`的`Token`(或者说`Index,Id`)将作为对应`vnode`进程的`id`标识。如下图所示

多个`partition`可以挤在一个物理节点上。但是怎么挤是有严格要求的,我觉得看懂这个图的关键在 于:每一种颜色代表同一个物理节点;各个颜色按照固定的顺序循环。这保证了任意两个相邻的 `partition`肯定不在同一个物理节点上。因此`Dynamo Preference List`上的节点不会是同一个物理节点,也即一份数据的多份副本不会在同一个物理节点上了。(这里一个`物理节点指一个erlang虚拟机`,而不是一个实际的计算机)
    
`Riak Core`的基本原理是,通过一致性`hash`算法,系统要处理的数据(例如,KV要存储的业务数 据,或者要处理的用户请求会话)会被`riak_core`随机的均匀分布在环上的各个分区中, 对每个数 据的处理由该分区上的`vnode`进程负责。
由于`hash`算法的特点,当我们要对某个数据集进行处理时,这个数据集会随机分布个不同的`partition`上,所以本质上是个数据并行的处理方式。

- 2.`Riak Core`的设计:`partition`和`vnode`

按照`dynamo`的设计思想,要处理的数据将会随机均匀的分布在`dynamo ring`上,`Riak Core`进一步将这些数据又以`partition`为基本单元组织起来。对数据的处理将以`partition`为单元,通过`vnode`进程进行处理。数据的处理(或者叫服务)又有很多种,每类服务对应着一类`vnode`进程。

显然,`partition`是整个分布式系统并发、复制和容错的基本单元:以`partition`为单元进行数据并发 处理;复制以`partition`为单元进行;容错也是如此:出错也以`partition/vnode`为单元出错。

对于基于`Riak Core`的分布式应用系统开发来说,`vnode`是最重要的概念,简单的说,每个`vnode`进程负责一份`partition`上数据的处理,数据处理逻辑由用户负责实现。

一份`partition`可以有多个`vnode`进程管理,所以上图中`a single partition/vnode`应该是`a single partition with its assorted vnodes`。

**(用`riak_core_node_watcher:services()`可以察看应用系统中有哪些`vnode`服务)。**

例子写道例如,对`riak`这个`NoSQL`数据库来说,最重要的服务是存储,在`riak_kv_vnode`模块中实现。模块实现了`riak_core_vnode`接口(实现了`riak_core_vnode behaviour`的回调函数)。

- 3.此外`riak`还有两类重要的服务: 

`riak`的`mapreduce`基于`riak_pipe`实现,其`vnode`对应着`riak_pipe_vnode`模块。 
    
`riak`还提供了**二级索引搜索(`riak_search`)**服务,其`vnode`对应着`riak_search_vnode`模块。

而一个实际的应用系统可能在功能上要比`NoSQL`数据库更多,它需要根据业务提供各种各样的服务,因此在`vnode`种类上就显得丰富多彩。比如`rts`实时日志统计这样一个例子就有两类`vnode: riak_core_entry_vnode`和`riak_core_stat_vode`,前者记录它负责的那份`partition`的所有数据, 后者统计它负责的那份`partition`的所有数据。


## riak core 依赖注入 loC

#### riak_core_vnode behaviour 用户处理逻辑的实现接口

`Riak Core`为提供了一个`riak_core_vnode`回调模块作为用户实现业务逻辑的接口。

`riak_core_vnode`有两个作用:

- 1.作为erlang behaviour提供回调函数实现接口,用户在回调函数中实现业务逻辑;

- 2.作为容器进程运行实现上述接口的用户模块,具体来讲它将作为一个基于gen_fsm的有限状态机进程运行。
也就是说,`riak_core_vnode`是作为一个容器以`erlang`进程的形式运行,而用户的业务逻辑 栖身于容器中。(BTW:`gen_server/gen_fsm`都是这种设计和运行模式。)
我想这是一种典型的`IoC`(依赖注入)设计:系统将用户实现的功能注入到`Riak Core`系统 中,由此构建出基于`Riak Core`的分布式应用系统。困难的`dynamo`分布式实现交给`Riak Core`完成,用户只关心业务逻辑的实现就行了。


#### 业务逻辑的实现和其他

主要是实现`riak_core_vnode behaviour`的接口回调函数。除了业务逻辑的实现,还设计到 数据的转移等。

`Riak Core`将`partition`上的数据处理操作抽象出来,由用户通过`riak_core_vnode behaviour`的回调函数实现,`vnode`的回调函数分为以下几类:

- `vnode`进程生命周期管理,在`vnode`进程诞生和毁灭时将被调用的回调函数`init/1`, `terminate/2`

- 对`vnode`进程负责的`partition`上数据的处理,当传来对该`partition`上的数据继续处理的命 令时,由回调函数`handle_command/3`负责处理。例如,如果对数据的处理是存储,那么无非是`get/put`之类的命令,这些命令的格式(协议)都是由用户自己定义并实现的。

- 当集群中增/减物理节点时,或者某个宕掉的物理节点恢复时,就需要对`partition`进行`handoff`的处理,其核心是对数据的搬运,它包括一组回调函数
`(is_empty/1, delete/1, handle_handoff_command/3, handoff_starting/2, handoff_cancelled/1, handoff_finished/2, handle_handoff_data/2, handle_handoff_data/2, encode_handoff_item/2 )`

- 对进程错误的处理:与该`vnode`相关的进程如果出错会调用回调函数`handle_exit`

- `Covering`:覆盖整个`ring`空间数据的特殊命令,例如对一段连续范围的搜索命令。某个连 续的数据集在`hash`后有可能会落到所有的`partition`。

#### 业务逻辑的注入过程

我们的分布式数据处理系统作为一个`OTP application`发布。在`application`中实现了处理系 统的启动,以及业务逻辑的注入。

`application`启动时会启动`supervisor`,后者监控着`riak_core_vnode_master`子进程 (`riak_core_vnode_master`是一个`gen_server`),参数配制成实现用户逻辑的定制`vnode`模块,从而实现业务逻辑注入。

> riak_core_vnode_master可看成是运行着vnode的容器.

【业务逻辑的注入】

``` erlang

    -module(rts_sup).
    -behaviour(supervisor).
    %% API -export([start_link/0]).
    %% Supervisor callbacks -export([init/1]).
    start_link() ->
        supervisor:start_link({local, ?MODULE}, ?MODULE, []).
    init(_Args) ->
        VMaster = { rts_vnode_master,
        {riak_core_vnode_master, start_link, [rts_vnode]}, permanent, 5000, worker,     [riak_core_vnode_master] },
        Entry = { rts_entry_vnode_master,
        {riak_core_vnode_master, start_link, [rts_entry_vnode]},
        permanent, 5000, worker, [riak_core_vnode_master] },
        Stat = { rts_stat_vnode_master,
        {riak_core_vnode_master, start_link, [rts_stat_vnode]},
        permanent, 5000, worker, [riak_core_vnode_master] },
        {ok,{{one_for_one, 5, 10},
            [VMaster, Entry, Stat]}}.
```

有几种执行特定分区上业务逻辑的API方法,都是`riak_core_vnode_master`模块提供的,这 些函数的参数都是三个,分别是: **IdxNode, Msg, VMasterMod**

- 1.同步发送

`sync_command(IndexNode, Msg, VMaster)`

- 2.异步发送

`riak_core_vnode_master:sync_spawn_command(IndexNode, ping, rts_vnode_master)`

还有一个比较特殊`command`函数,它有两种
一种一上述两类相似,直接向vnode同步发送命令: `riak_core_vnode_master:command(IdxNode,
{entry, Client, Entry} = Msg, rts_entry_vnode_master = VMaster)`.

一种向`PreList`里的所有节点发送`command`命令: `command(Preflist, Msg, VMaster)`
`IdxNode`是`{Index, Node}`这样的`tuple`,其中`Index`是`partition`分区的`id`号,`Node`是分区所 在的节点,知道了`Node`和`rts_entry_vnode_master`,就能得到`vnode`进程`Pid`,通过发送事件调用相关业务逻辑:
    `gen_fsm:send_event(Pid, make_request(Msg, Sender, Index));`

*IndexNode的计算*:

- 用`riak_core_util:chash_key/1`函数计算hash值,参数是一个含两个二进制数据的`tuple`
`DocIdx = riak_core_util:chash_key({<<"ping">>, term_to_binary(now())}),`

- 通过哈希值得到`Preflist,Preflist`的长度可以指定:
`PrefList = riak_core_apl:get_primary_apl(DocIdx, 1, rts).`

最后得到`IndexNod [{IndexNode, _Type}] = PrefList`

*BTW: 应用启动过程*

``` erlang
- module(rts_app). 
-behaviour(application).
%% Application callbacks -export([start/2, stop/1]).
start(_StartType, _StartArgs) -> 
    case rts_sup:start_link() of
        {ok, Pid} ->
            ok = riak_core:register_vnode_module(rts_vnode), 
            ok = riak_core_node_watcher:service_up(rts, self()),
            ok = riak_core:register_vnode_module(rts_entry_vnode),
            ok = riak_core_node_watcher:service_up(rts_entry, self()),
            ok = riak_core:register_vnode_module(rts_stat_vnode), 
            ok = riak_core_node_watcher:service_up(rts_stat, self()),
            EntryRoute = {["rts", "entry", client], rts_wm_entry, []}, 
            webmachine_router:add_route(EntryRoute),
        {ok, Pid}; {error, Reason} ->
            {error, Reason} 
    end.

stop(_State) -> 
    ok.
```


## Riack Core 系统的数据的分布和处理步骤

从原理上讲,`Riak Core`通过`hash`算法将数据随机均匀的分布在一个环上,数据的hash值也 就是在环上的位置(源码中常用`Index`表示),知道了`Index`就知道了对应的分区`partition`, 知道了分区就知道了对应的管理节点。知道了这个节点我们可以向其发送对应的处理命令:如 取出这块数据、修改这块数据、对这块数据进行计算等等。
在这样的系统上处理数据的步骤如下:

- 1)计算数据在环上的位置;
- 2)计算这个位置对应分区`partition`的管理节点; 
- 3)向这个(些)节点发送处理命令

对环上分布在不同节点上的这些数据的操作可以并发的进行,因此`Riak Core`本质上还是一个 数据并行的分布式系统。

以上是客户使用时的处理过程(或步骤)。而开发人员在基于`Riak Core`构建分布式系统时主 要面临两个问题:

- 1.数据如何分布:Riak Core通过某种hash算法将数据随机均匀的分布在一个环上。选取 那种hash算法,如何hash,可以由开发人员自由选择。例如,如果数据有唯一key,那 么可以对key进行hash,具体来讲,riak这个kv 存储系统默认是对bucket+key做 hash。(当然riak也可以对不同的bucket配置不同的hash方式,包括hash算法和要 hash的数据)
- 2.数据如何处理:也即节点能处理哪些命令,以及这些命令的实现。对数据的处理涉及到 系统的业务逻辑,这得由开发人员自己实现。例如对riak来说,主要的业务逻辑就是存 储和查找了。

**数据如何分布**

  数据的分布是指如何将给定的数据映射到环上的位置(`Index`)。数据如何在环上分布可以由 我们自己自行决定,本文为了表述方便称之为“数据分布的哈希策略”,这包括两方面:

- 1.哈希算法的选择:Riak Core默认的hash算法是SHA算法,当然我们也可以选择自己的 hash算法,不过实在没有这个必要。
- 2.哈希的对象:hash函数接收一个对象参数,这个参数是一个含两个二进制数据的term;我们根据数据的特点自行确定term的组合方式,例如对于riak这样的key-value数据库,这个 term的值就是这样子: `{<<"bucket", "key">>}`
实际上Riak Core提供的了两个hash函数,一个叫`chash_std_keyfun`,一个叫 `chash_bucketonly_keyfun`,缺省的是`chash_std_keyfun`函数。相同的是它们都采用了SHA 哈希算法,不同的是哈希对象的选取:

``` erlang
chash_std_keyfun({Bucket, Key}) -> chash:key_of({Bucket, Key}).
chash_bucketonly_keyfun({Bucket, _Key}) -> chash:key_of(Bucket).
```

可以看到,`has`h的对象不同,一个对`{Bucket,Key}`整体做hash,一个只对`Bucket`做hash, 他们都调用的`key_of`使用的是`SHA哈希算法`:

``` erlang
-spec key_of(ObjectName :: term()) -> index().
key_of(ObjectName) ->
crypto:sha(term_to_binary(ObjectName)).
```

这些工作是通过Riak Core提供助手模块`riak_core_util`中的函数`chash_key`进行,一般在交给 `vnode_master`之前要预先计算好了数据在`ring上`的位置(Index)。

``` elixir
@spec chash_key(BKey :: {binary(), binary()}) -> chash:index() 

chash_key({Bucket,Key}) ->
BucketProps = riak_core_bucket:get_bucket(Bucket),
{chash_keyfun, {M, F}} = proplists:lookup(chash_keyfun, BucketProps), M:F({Bucket,Key}).
```

代码不是很直观,原因是中间插入了额外的逻辑,为了实现不同`bucket`可以有不同的`hash`策略。
可以看到`chash_key`不做具体的`hash`计算,它只是调用了配置文件中设定的`hash`函数(用红色标注),换句话,我们可以自定义自己的hash函数,这个函数有唯一的参数,这个参数也是 
`{Bucket::binary(), Key::binary()}`

- **数据的处理步骤 在解决了数据如何分布的问题后,再来看看如何使用:**

  数据处理步骤中对应的第1步,计算数据在环上的位置,Riak Core提供了对应的API,及函数 `riak_core_util:chash_key/1`.

 以riak这个NoSQL数据库为例,它是通过对数据所在的bucket以及数据的key做hash计算得 到数据的分布节点的(即默认的

 `chash_std_keyfun):`
 
 `BKey={Bucket, Key}`

 `DataIdx = riak_core_util:chash_key(BKey),` `DataIdx`就是根据数据的分布策略得到的该数据在环上的位置。

  数据处理步骤中对应的第2步,计算这个位置对应分区`partition`的管理节点: 根据环上位置就可以得到对于的节点:
`PrefList = riak_core_apl:get_primary_apl(DataIdx, 1, rts)`,
`IdxNode = hd(PrefList)`, % 数据分布的第一个节点

 数据处理步骤中对应的第3步, 向这个(些)节点发送处理命令 知道了数据分布的节点,就可以给该节点发送命令了。
`PrefList = riak_core_apl:get_primary_apl(DataIdx, 1, rts),`
`IdxNode = hd(PrefList),` % 数据分布的第一个节点 `riak_core_vnode_master:command(IdxNode, ...` % 在数据分布节点上进行数据处理

一个完整的例子:`rts`这个例子中的`ping`,使用的是根据当前时间数据计算出分布的节点,然 后

- 1.`DataIdx = riak_core_util:chash_key({<<"ping">>, term_to_binary(now())}),`

- 2.`PrefList = riak_core_apl:get_primary_apl(DataIdx, 1, [i]rts[/i]),`

- 3.`IdxNode = hd(PrefList),` % 数据分布的第一个节点

- 4.`riak_core_vnode_master:command(IdxNode, ...` % 在数据分布节点上进行数据处理

***这样的效果是每次ping都ping到不同的节点上。***
## 业务逻辑的实现： 数据如何处理

重点是数据如何处理:`Riak Core`提供了一个统一的接口以控制分布在ring上的数据的计算(操作)。

***RiakCore的数据控制接口:***

如前所述,每类`vnode`提供了一套服务,每个服务由在各个`partition`上的`vnode`进程组成, 这些进程实际分布在各个物理节点上。对于每一个物理节点,每类服务会有一个 `riak_core_vnode_master`进程提供统一的接口。实际上`riak_core_vnode_master`进程的主要作用是将`请求转发给对应的vnode进程进行处理`,无论这个`vnode`进程在`哪个物理节点上`。

`riak_core_vnode_master`模块是一个`gen_server`的实现,它对外提供了一套API。
引用 用`OO`打个比方,如果将`riak_core_vnode_master`比作`OO`中的一个类,这些API相当于这个 类的`静态公共方法`,这些`静态公共方法`提供控制这个类的对象的`初始化、以及这些对象的控制等`,也就是说所有与这些对象打交道的操作统一通过这些静态公共方法进行。

我们主要通过`riak_core_vnode_master`模块提供的API控制集群内的所有`vnode`进程,从而 对外提供数据处理服务,这些API有:

- `start_vnode(VNodeModule)`函数:
    
启动`vnode_master`进程(一个`gen_server`进程) `并注入用户逻辑`,用户逻辑在`VNodeModule`参数指定的模块中实现。通过这种方式在启动时将用户的业务逻辑注入进来(类似IoC的构造器注入); 

- `get_vnode_pid(PartitionId, VNodeModule)`函数:

获取某个`partition`上某类服务的`vnode`进程,`如果不存在则调用riak_core_vnode_sup`创建之(也就是说`vnode`进程是***`懒加载`***,不过全部挂在`riak_core_vnode_sup`这个`supervisor`树下); 
- 用`command/3,sync_command/3,sync_spawn_command/3`:会将请求命令 (`command`)发给对应的`vnode`进程,最后调用用户实现的`vnode`模块中的 `handle_command`回调函数处理;

- `converage/5`函数:稍候研究 `to be continue...`

可以看到`riak_core_vnode_master`的主要任务是转发用户请求,可以异步转发,也可以同步转发。

如何转发:根据请求的数据所在的`partition`,可以算出`partition`所在的物理节点和对应的vnode进程Pid,知道了物理节点和vnode进程的pid后就可以向此物理节点上的vnode进程直接发送消息。
不过,`vnode`进程的生成不由`vnode_master`直接负责,所有`vnode`进程的创建实际上由一个专门的
`riak_core_vnode_sup`模块负责(这可模块实际上是一个`simple_one_for_one`策略的`supervisor`,这种启动策略使它只能管理同一类erlang进程,不过它的特点是能动态的创建无数的相同类型的子进程,见`supervision principles`)。vnode进程本质上是一个`gen_fsm`状态机(可以看到有一个`active`状态)

***数据处理:一个完整的例子***

`try-try-try`的例子代码要简单许多,这个例子是随机ping一个分区vnode,返回结果是该分区 的ID和物理节点,整个过程如下:

``` erlang
ping() ->
    DocIdx = riak_core_util:chash_key({<<"ping">>, term_to_binary(now())}), 
    
    PrefList = riak_core_apl:get_primary_apl(DocIdx, 1, rts),

    [{IndexNode, _Type}] = PrefList,

    riak_core_vnode_master:sync_spawn_command(IndexNode, ping, rts_vnode_master).
```

`riak_kv`中的例子要复杂很多,不过基本过程还是类似的.

***其它***

在每个物理节点上会为`每一类vnode`启动一个`riak_core_vnode_master`进程,该进程控制这个物理节点上的所有`同类vnode进程`(通过上面提到的API)。所有这些 `riak_core_vnode_master`进程的注册遵循一套约定的命名规则:
  **`master`进程的注册名就是该类`vnode`的模块名字加上尾缀"`_master`"。例如应用系统实现了一类`vnode`,其模块叫 `rts_stat_vnode`,对应的`riak_core_vnode_master`进程名就在erlang中注册 为“`rts_stat_vnode_master`”。**

可以通过`riak_core:vnode_modules()`察看当前有多少`vnode`模块。(它实际上反映了通过 `riak_core:register_mod/3`注册的模块,该函数将`vnode`模块信息放置在`application`环境 中。许多函数调用了它,例如`riak_core:register_vnode_module/1`)

`vnode`进程是懒加载的,由`riak_core_vnode_sup`这个`supervisor`负责动态创建`vnode`进程 (可以通过`supervisor:which_children(riak_core_vnode_sup)`.察看当前的所有 `vnode_worker`进程)。

浏览`rts`实时日志统计的代码有助于了解基于`riak_core`的应用系统的工作方式。 应用系统的功能由`rts`这个`application`实现,在`rts_app`这个`application behaviour`中可以看到它启动了`rts_sup`,成功后注册了`3类vnode`,并作为服务启动查看图片附件

## 常用的API
### 常用API之与ring、节点有关的API

- `riak_core:vnode_modules()` 查询安装的vnode模块 
- `riak_core_ring_manager:get_raw_ring()` 获取整个ring环,包括partition及其节点
- `riak_core_apl:get_apl(HashKey, N, Service)` 得到HashKey对应的Preference List
- `riak_core_ring:preflist(HashKey, Ring)` 得到HashKey在环Ring上的Preference List

### 常用application管理API 
erlang本身提供的application模块有许多API函数可用:

- `aplication:loaded_applications/0` 察看当前有哪些应用已经装载 
- `aplication:which_applications/0` 察看当前有哪些应用已经启动

erlang以application组织系统,application由模块组成,多个共同组成一个系统。 已装载的`application`并不意味着**它已经启动**,这一般是在`rel`( 也即`script` )文件中指定

application又依赖其它applications,后者又依赖其它application。。。 例如
A依赖B、C:这意味着A要正常启动,B、C要先启动, B又依赖X、Y:A要正常启动,Y、Z要先启动, C又依赖Y、Z:C要正常启动,Y、Z要先启动,
以此类推。。。

一般在每个应用对应的`.app`文件中,有个`applications`的配置,其值是个列表(list),这个 list就是该应用依赖的其它应用。在系统发布时(release),发布工具会自动将这个列表里的所以依赖应用都启动。

如果不是作为发布(`release`),在开发过程中经常要`手动启动(application:start/1)`,但是这种启动是不会自动将列表中的依赖应用也启动的,会报如下错误:

``` erlang
1> application:which_applications().
[{stdlib,"ERTS CXC 138 10","1.18.1"},
{kernel,"ERTS CXC 138 10","2.15.1"}] 
2> application:start(xxx). {error,{not_started,lager}}
```

因此启动`xxx`应用之前要确保`xxx`的依赖应用`lager`先被启动(`application:start/1`),而`lager`应 用又依赖`compiler`和`syntax_tools`应用,因此都得按照依赖关系一一启动:

``` erlang
3> application:start(lager).
{error,{not_started,compiler}}
```

在应用的`app`文件中会有一个`applications`属性,列出所有的依赖应用。

``` erlang
4> application:start(compiler).
ok
5> application:start(lager). {error,{not_started,syntax_tools}} 
6> application:start(syntax_tools). ok
7> application:start(lager).
18:40:11.591 [info] Application lager started on node nonode@nohost 
ok
8> application:start(xxx).
18:40:13.739 [info] Application xxx started on node nonode@nohost 
ok
```

在实际开发过程中,这种依赖的一一启动是很枯燥的,`Riak Core`的`riak_core_util`模块提供了这种依赖应用的管理:
`riak_core_util:start_app_deps/1` 这个函数检查应用依赖的其它应用,并确保所有依赖应用被启动。判断依赖应用`:app`文件中的`applications`属性。


