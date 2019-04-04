---
title: erlang学习笔记
date: 2016-04-25 00:15:14
tags:
- Erlang
categories:
- Erlang

description: 好久没看过erlang了，趁现在博客的迁移，重温一下。
---


## `erlang` 之简单的`Diction`实现


最近在看学erlang ，看到了字典这个demo ，把程序Copy出来和大家分享一下

``` erlang
    -module (diction).  
    -export([new/0,lookup/2,add/3,delete/2]).  
    new()   ->  
        [].  
      
    lookup(Key , [{Key,Value}|Rest])    ->  
        {value,Value};  
    lookup(Key,[Pair|Rest])     ->  
        lookup(Key,Rest);  
    lookup(Key,[])  ->  
        undefined.  
    add(Key,Value,Diction)  ->  
        NewDict =   delete(Key,Diction) ,  
        [{Key,Value}|NewDict].  
      
    delete(Key,[{Key,Value}|Rest])  ->  
        Rest;  
    delete(Key,[Pair|Rest]) ->  
        [Pair|delete(Key,Rest)];  
    delete(Key,[])  ->  
        [].  
```

函数编程习惯之后，写起来也是挺爽的意见事，基本上都是递归的思想。

## `erlang` 之简单密码加密 

> 这些程序主要是来之 连城 翻译的一个书里面的代码

``` erlang
    -module(encode).  
    -export([encode/2]).  
    encode(Pin.Password)    ->  
        Code = {nil,nil,nil,nil,nil,nil,nil,nil,nil,  
            nil,nil,nil,nil,nil,nil,nil,nil,nil,  
            nil,nil,nil,nil,nil,nil,nil,nil},  
        encode(Pin,Password,Code).  
    encode([],_,Code)   ->  
        Code ;  
    encode(Pin,[],code) ->  
        io:format("Out of Letters~n",[]);  
      
    encode(H|T,[Letter|T1],Code)    ->  
        Arg = index(Letter) +1 ,  
        case element(Arg,Code) of   
            nil ->  
                encode (T,T1,setelement(Arg,Code,index(H)));  
        _->  
            encode ([H|T],T1,Code)  
        end.  
      
    index(X)    when X >= $0 ,X =< $9 ->  
            X - $0;  
    index(X)    when X>=$A , X =< $Z  ->  
            X - $A.  
```

##  erlang 简单的树操作 

```erlang
-module(tree).  
-export([test1/0]).  
lookup(Key,nil) ->  
    not_found;  
lookup(Key,{Key,Value,_,_}) ->  
    {found,Value};  
lookup(Key,{Key1,_,Smaller,_}) when Key < Key1   ->  
    lookup(Key,Smaller);  
lookup(Key,{Key1,_,_,Bigger})   when Key > Key1 ->  
    lookup(Key,Bigger).  
  
insert(Key,Value ,nil)  ->  
    {Key,Value,nil,nil};  
insert(Key,Value,{Key,_,Smaller,Bigger})    ->  
    {Key,Value,Smaller,Bigger}  ;  
insert(Key,Value,{Key1,V,Smaller,Bigger})   when Key < Key1 ->  
    {Key1,V,insert(Key,Value,Smaller),Bigger};  
insert(Key,Value,{Key1,V,Smaller,Bigger})   when Key > Key1  ->  
    {Key1,V,Smaller,insert(Key,Value,Bigger)}.  
write_tree(T)   ->  
    write_tree(0,T).  
write_tree(D,nil)   ->  
    io:tab(D),  
    io:format('nil',[]);  
write_tree(D,{Key,Value,Smaller,Bigger})    ->  
    D1 = D +4 ,  
    write_tree(D1,Bigger),  
    io:format('~n',[]),  
    io:tab(D),  
    io:format('~w ==> ~w~n',[Key,Value]),  
    write_tree(D1,Smaller).  
  
test1() ->  
    S1=nil,  
    S2=insert(4,joe,S1),  
    S3=insert(12,fred,S2),  
    S4=insert(3,jane,S3),  
    S5=insert(7,kalle,S4),  
    S6=insert(6,thomes,S5),  
    S7=insert(5,rickard,S6),  
    S8=insert(9,susan,S7),  
    S9=insert(2,tobbe,S8),  
    S10=insert(8,dan,S9),  
    write_tree(S10).  
```

代码不多，`erlang`写算法......呵呵呵呵呵呵


## erlang 并发编程 

最近上班比较忙，没时间学习erlang ，实在对不起自己啊，以前一直在找erlang相关的教程，终于找到一个了，这个网站是前几天才开始运行的，以后的文章可能都是来自于那里，网站是`http://www.erlang-cn.com` ，大家忙没事多学习！大笑

- 并发编程一

```erlang
-module(tut15).  
-export([start/0,ping/2,pong/0]).  
ping(0,Pong_PID)    ->  
    %%想对方发送退出信号  
    Pong_PID ! finished,  
    io:format("ping finished~n",Pong_PID);  
ping(N,Pong_PID)    ->  
    Pong_PID ! {ping,self()},  
    receive   
        pong    ->  
            io:format("Ping receive pong ~n",[])  
    end,  
    %%继续接受信息，直到 N == 0  
    ping(N-1,Pong_PID).  
  
pong()  ->  
    receive  
        finished    ->  
            io:format("Pong finished~n",[]);  
        {ping,Ping_PID} ->  
            io:format("Pong received ping ~n",[]),  
            Ping_PID!pong,  
            %%再次等待对方的信息，直到信息为ffinished  
            pong()  
        end.  
start() ->  
    %% 开启一个进程，用来等待其他进程的信息  
    Pong_PID = spawn(tut15,pong,[]),  
    %%开启一个进程，用来发送信息  
    spawn(tut15,ping,[3,Pong_PID]),  
    io:format("Main Process Exit~n",[]). 
```

进程的标识 ，我们还有一种更加灵活的方法来标记她那就是使用`register`函数

```erlang
-module(tut16).  
-export([start/0,ping/1,pong/0]).  
  
ping(0) ->  
    pong ! finished,  
    io:format("ping finished ~n",[]);  
ping(N) ->  
    %% send the message to pong proccess  
    pong ! {ping,self()},  
  
    receive   
        pong ->  
            io:format("Ping received pong ~n",[])  
    end ,  
    ping(N-1).  
  
pong()  ->  
    receive   
        finished    ->  
            io:format("Pong finished~n",[]);  
        {ping,Ping_PID} ->  
            io:format("Pong received ping~n",[]),  
            Ping_PID ! pong,  
            pong()  
    end.  
  
start() ->  
    register(pong,spawn(tut16,pong,[])),  
    spawn(tut16,ping,[3]).  
```

`register`中的`pong`就标识了`spawn(tut16,pong,[])`这个进程，这个`ping（）`函数只要一个`N`就行了，`pong`可以看作是进程之间共享的变量.

## erlang 之 echo 服务器 

简单实现了一个echo 服务器

- echo_server1

```erlang
    -module(echo).  
    -export([start/0,loop/0]).  
      
    start() ->  
        Pid = spawn(echo,loop,[]),  
        Pid ! {self(),'Hello Word'},  
        receive   
            {Pid,Msg}   ->  
                io:format('~w~n',[Msg])  
        end,  
        Pid ! stop.  
      
    loop()  ->  
        receive   
            {FromOther,Msg} ->  
                io:format("~w~n",[Msg]),  
                FromOther!{self(),'Loop Proccess Send to You !'},  
                loop();  
            {stop}  ->  
                %%io:format('~w~n',[loop_stop]),  
                true  
        end.  
```

Output：

```erlang
29> echo:start().  
'Hello Word'  
'Loop Proccess Send to You !'  
stop  
30> 
```

其中 stop 是主进程的返回值

- echo_server2

 这是一个简单用于等待外部信息的echo server

```erlang
    -module(echo_server).  
    -export([start/0,print/1,stop/0,loop/0]).  
      
    start() ->  
        Pid = spawn(echo_server , loop , []),  
        register(sub1,Pid),   
        {ok,Pid}.  
      
    loop()  ->  
        receive   
            {print,A}   ->  
                io:format("~p.~n",[A]),  
                loop();  
            stop    ->  
                true;  
            Other ->  
                io:format("~p~n",[Other]),  
                loop()   
        end.  
      
    print(A)    ->  
        sub1 ! {print,A},  
        true .  
    stop()  ->  
        sub1 ! stop ,  
        %%   
        true .  
```

## timeout 的简单使用 


**今天晚上有点晚了，不过还是坚持每一天写一个程序！**

下面的时超时器 ：

建设`A`要想`db`进程发送一个信息，然后在规定的时间内等待消息的返回，那么`A`可以设置一个超时器，注意的是在发送消息之前，得先清空消息队列，要不然等译接到的消息可能`db`还没发送之前的了.

``` erlang
    read(Key)   ->  
        flush(),  
        db | {self(),{read,Key}},  
        receive   
            {read,R}    -> {ok,R};  
            {error,Reason}  -> {error,Reason}  
        after 1000  ->   {error,timeout}  
        end.  
      
    flush() ->  
        receive   
            {read,_}    -> flush();  
            {error,_}   ->flush()  
        after 0 ->ok   
    end.  
```

## Erlang进程生成测试 

```erlang
    -module(myring).  
    -export([start/1,start_proc/2]).  
    start(Num)  ->  
        start_proc(Num,self()).  
    start_proc(0,Pid)   ->  
        receive   
            ok -> ok   
        end,  
        Pid ! ok ;  
    start_proc(Num,Pid) ->  
        NPid = spawn(?MODULE,start_proc ,[Num-1,Pid]),  
        NPid ! ok ,  
        receive   
            ok -> ok ,  
            io:format("~w~n",[Num])  
        end.  
```
进程退出时会返回ok。

## `erlang`之时钟 

今天来看一下`erlang`中的时钟如何实现的：

```erlang
    -module(timeout).  
    -export([sleep/1,flush_buffer/0]).  
      
      
      
    %%%睡眠函数  
    sleep(Time) ->  
        receive   
            after Time ->  
            true  
        end.  
      
    %%%清空邮箱  
      
    flush_buffer()  ->  
        receive   
            AnyMessage  ->  
                flush_buffer()  
            after   0   ->  
                true  
        end.  
      
    %%% 消息优先级的实现  
    %% 函数priority_receive会返回邮箱中第一个消息，除非有消息interrupt发送到了邮箱中，此时将返  
    %%回interrupt。通过首先使用超时时长0来调用receive去匹配interrupt，我们可以检查邮箱中是否已经有了  
    %%%这个消息。如果是，我们就返回它，否则，我们再通过模式AnyMessage去调用receive，这将选中邮箱中的  
    %%第一条消息。  
    priority_receive()  ->  
        receive  
            interrupt   ->  
                interrupt  
            after   0   ->  
                receive  
                    AnyMessage  ->  
                        AnyMessage  
                end  
        end.  
```

上面主要是睡眠 和清空“邮箱” ，还有就是优先级的简单实现.

下面再来一段吧：

```erlang
    -module(timer).  
    -export([timeout/2,cancel/1,timer/3]).  
      
    timeout(Time,Alarm) ->  
        spawn(timer,timer,[self(),Time,Alarm]).  
      
      
    cancel(Timer)   ->  
        Timer ! {self(),cancel}.  
    timer(Pid,Time,Alarm)   ->  
        receive       
            {Pid,cancel}    ->  
                true  
        after   Time    ->  
            Pid ! Alarm  
        end.  
```

`shell`中演示一下：
```erlang
    13> Pid=self(),  
    13> timer:timer(Pid,1000,hellword).  
    hellword  
    14>   
```

## a simple erlang process pool analysis

<center>![](http://laohanlinux.github.io/images/img/blog/erlang/pool/ppoll.jpg)</center>

这是一个简单的`erlang`进程池分析，是`learn you some erlang for Great Good`里面的一个`example`，详细的内容可到官网查看！

- 实现原理

 这个的例子的实现原理官网都有比较详细的说明，主要模块在`ppool_serv`中，`ppool_serv`是一个`gen_server behaviour`, 而`ppool_sup`是一个`one_for_all`的策略,如果`ppool_serv`或者`worker_sup`出现问题，彼此也没有存在的必要了。

这里`ppool_serv`和`worker_sup`的实现，使用了一个简单的技巧，因为`worker_sup`不是`ppool_sup`直接调用生成的，它是由`ppool_serv`控制生成的：

```erlang

%% Gen server
init({Limit, MFA, Sup}) ->
    %% We need to find the Pid of the worker supervisor from here,
    %% but alas, this would be calling the supervisor while it waits for us!
    self() ! {start_worker_supervisor, Sup, MFA},
    {ok, #state{limit=Limit, refs=gb_sets:empty()}}.
```

`woker_sup`由`ppool_serv`自己在`init`函数中，发给自己一个`Message`，然后在回调函数中才生成：

```erlang
handle_info({start_worker_supervisor, Sup, MFA}, S = #state{}) ->
    {ok, Pid} = supervisor:start_child(Sup, ?SPEC(MFA)),
    link(Pid),
    {noreply, S#state{sup=Pid}};
```

如果他们一起直接生产，那么会产生死锁，
<center>![](http://laohanlinux.github.io/images/img/blog/erlang/pool/ppool_1.png)</center>

当然，他这里的生成顺序，可以自己修改一下，也不会出现死锁。

`gen_serv`的主要数据结构

```erlang
-define(SPEC(MFA),
        {worker_sup,
         {ppool_worker_sup, start_link, [MFA]},
          temporary,
          10000,
          supervisor,
          [ppool_worker_sup]}).

-record(state, {limit=0,
                sup,
                refs,
                queue=queue:new()}).
```

`?SPEC(MFA)`, 这里的`MFA`指明一类`Task`，所以同一个`ppool_worker_sup`,不会有不同类型的`Task`，它的策略也是`simple_one_for_one`.

在这个例子中，使用了一个`gen_server -- nnager module`作为`Task`，这个`Task`的参数为：`{Task, Delay, Max, SendTo}`， `Task`标示任务名字，`Delay`作为超时时间，只是标示这个任务是有超时限制的，也是一个调试技巧，`Max`为最大超时次数，`SendTo`用来发送信息给回调进程，这个进程可以是`shell`， 如果是`shell`，`flush()`就会收到信息。

`record`用来标识一些主要的信息，`Limit`为进程池的大小限制，`sup`开始为`ppool_sup`的`pid（）`，在生成`woker_sup`进程后，就变成`worker_sup`的进程`pid（）`，因为`ppool_serv`的主要交流对象还是`worker_sup`和`worker(Task)`； `refs（gb_set）`为`woker`的进程链接，这样可以在`worker`进程`down`掉或者`done`时，从线程池中剔除掉；`queue`为任务队列，当任务大于`limit`时，就把多余的任务放到`queque`中，等到进程池有空闲时，就从中`pop`出任务，接着处理。

这里有些局限的地方：
- 每次的任务都是新建的进程去处理，就是说进程的生命周期跟任务的生命周期是一样的，可以把进程跟任务分离出来，让进程不随任务的结束而结束（当然这的任务就不要是`gen_server`,`gen_fsm`这些，因为这些也是`spawn`出来的进程），这样进程开销理论上一次初始化就行了，虽然进程在`erlang`中开销比较少；
- 队列没有大小限制


## PoolBoy
 
source code ：

https://github.com/devinus/poolboy

![](http://laohanlinux.github.io/images/img/blog/erlang/pool/poolboy.png)

- Checkout

```erlang

ready({checkout, Block, Timeout}, {FromPid, _}=From, State) ->
    #state{supervisor = Sup,
           workers = Workers,
           monitors = Monitors,
           max_overflow = MaxOverflow} = State,
    case queue:out(Workers) of
        {{value, Pid}, Left} ->
            Ref = erlang:monitor(process, FromPid),
            true = ets:insert(Monitors, {Pid, Ref}),
            NextState = case queue:is_empty(Left) of
                true when MaxOverflow < 1 -> full;
                true -> overflow;
                false -> ready
            end,
            {reply, Pid, NextState, State#state{workers=Left}};
        {empty, Empty} when MaxOverflow > 0 ->
            {Pid, Ref} = new_worker(Sup, FromPid),
            true = ets:insert(Monitors, {Pid, Ref}),
            {reply, Pid, overflow, State#state{workers=Empty, overflow=1}};
        {empty, Empty} when Block =:= false ->
            {reply, full, full, State#state{workers=Empty}};
        {empty, Empty} ->
            Waiting = add_waiting(From, Timeout, State#state.waiting),
            {next_state, full, State#state{workers=Empty, waiting=Waiting}}
    end;
```

`Checkout`出一个`worker`从`worker Queue`中，如果有,则`monitor（ets）`这个`worker`，然后根据队列的容量和`MaxOverflow`的值来确定下一状态为`full`，`overflow`，`ready` （`ready + overflow <= full`）;如果没有，而`MaxOverFlow`的值大于`0`，则新建一个`worker`，并将其加入`monitor`，最后重置状态项`worker = empty, overflow = 1`；如果没有，并且`MaxOverflow` 小于`1`， `Block == false`，则`{reply, full, full, State#state{workers=Empty}}`;如过没有，`MaxOverflow < 1`, `Block == true`,则 `{next_state, full, State#state{workers=Empty, waiting=Waiting}}`。

- Checkin

```erlang
ready({checkin, Pid}, State) ->
    Monitors = State#state.monitors,
    case ets:lookup(Monitors, Pid) of
        [{Pid, Ref}] ->
            true = erlang:demonitor(Ref),
            true = ets:delete(Monitors, Pid),
            Workers = queue:in(Pid, State#state.workers),
            {next_state, ready, State#state{workers=Workers}};
        [] ->
            {next_state, ready, State}
    end;
```

从`Monitor`中剔除对应的`worker`，然后回收到`worker queue`中去。
状态转变

- 状态转变的计算：

`worker_queue_size`(当前`size`) + `maxoverflow` 。

`ready`只与当前`worker_queque_size`有关，`overflow` 和`worker_queue_size(0)`和`maxoverflow>0`有关，`full`和`work_queue_size(0)`, `overfllow = maxoverflow`有关。
- `worker`的来源

所有的`worker`要么在初始化时创建平；要么调用`checkout`时，经过`poolboy`创建，但此时创建的`worker`没有进到`worker queue`，要想进到`worker queue`，只能调用`checkin`。
`work pid` 回收到`worker_queue`中

``` erlang
checkin_while_full --》{empty, Empty} when MaxOverflow < 1 ->；
```
