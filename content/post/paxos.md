---
title: paxos
date: 2016-03-30 21:04:29
tags: 
- paxos
categories: 
- Distribute System

description: 本文的内容源至--知行学社V.好吧，看完后，我还是一直半解...还是使用raft吧../?????

---

> 本文的内容源至--知行学社V.
> http://www.tudou.com/programs/view/e8zM8dAL6hM/

# Paxos 的理解困境
 
- Paxos究竟在解决什么问题？
- Paxos如何在分布式存储系统中应用？
- Paxos算法的核心思想是什么
    第一阶段在做什么
    第二阶段在做什么
 
# Paxos 用来确定一个不可变量的取值
- 取值可以是任何二进制数据
- 一旦确定将不再更改，并且可以被获取到(不可变性、可读性)
 
# 在分布式存储系统中应用Paxos
- 数据本身可变，采用多副本进行存储
- 多副本的更行操作序列[Op1,Op2,…,Opn]是相同的，不变的
- 用Paxos依次来确定不可变量Opi的取值（即第i个操作是神什么）
- 每次确定完Opi之后，让各个数据副本执行Opi，依次类推。
 
> Google的Chubby、Megastore 和Spanner都采用了Paxos来对数据副本的根性序列达成一致
 
# Paxos 希望解决的一直性问题

设计一个系统，来存储名称为var的变量
- 系统内部由多个Acceptor组成，负责存储和管理var变量。
- 外部有多个proposer机器任意并发调用API，想系统提交不同的var取值，var的取值可以是任意二进制数据
- 系统对外的API库接口为：`propose(var,V)`=> `<ok,f>` or `<error>`

## 系统需要保证var的取值满足一致性
- 如果var的取值没有确定，则var的取值为null
- 一旦var的取值被确定，则不可以被更改。并且可以一直获取到这个值

## 系统需要满足容错特性
- 可以容忍任意propose机器出现故障
- 可以容忍少数Accptor故障（半数一下）
 
暂时不考虑 网络分化 和 acceptor故障丢去var的信息。
 
# 确定一个不可变变量—难点
- 管理多个Proposer的并发执行
- 保证var变量的不可变性
- 容忍任意Proposer机器故障
- 容忍半数以下Acceptor机器故障
 
## 确定一个不可变量变量的取值----方案1
- 先考虑系统由单个Acceptor组成。通过类似互斥锁机制，来管理并发的proposer运行。
- Proposer首先向Acceptor申请Acceptor的互斥访问权，然后才能请求Acceptor接受自己的取值。
- Acceptor给Proposer发放互斥访问权，谁申请到互斥访问权，就接收到谁提交的取值。
- 让Proposer按照获取互斥访问权的顺序依次访问Acceptor
- 一旦Acceptor接受了某个Proposer的取值，则认为var取值被确定，其他Proposer不再更改
 
### 基于互斥访问权的Acceptor的实现
- Acceptor保存变量var和一个互斥锁lock
- Acceptor::pareprare():
加互斥锁，给予var的互斥访问权，并返回var当前的取值f
- Acceptor::release():
解互斥锁，收回var的互斥访问权
- Acceptor::accept(var,V):
如果已经加锁，并且var没有取值，则设置var为V。并且释放锁

### Propose(var,V)的两阶段实现
- 第一阶段：通过Acceptor::prepare获取互斥访问权和当前var的取值，如果不能返回<error>（锁被别人占用）
- 第二阶段：根据当前var的取值f，选择执行
如果f为null，则通过Acceptor::accept（var,V）提交数据V
如果f不为空，则通过Acceptor::release()释放访问权，返回<ok,f>
 
通过`Acceptor`互斥访问权让`Proposer`序列运行，可以简单的实现`var`取值的一致性

>Proposer在释放互斥访问权之前发生故障，会导致系统陷入死锁,所以方案一不能容忍任意Proposer机器故障
 
 
## 确定一个不可变变量的取值----方案2

引入抢占式访问权
- `Acceptor`可以让某个`Proposer`获取到的访问权失败，不再接收它的访问，之后，可以将访问权发给其他`Proposer`，让其他`Proposer`访问`Acceptor`
- `Proposer`向`Acceptor`申请访问权时指定编号`epoch`（越大的`epoch`越新），获取到访问权之后，才能向`Acceptor`提交取值。
- `Acceptor`采用喜新还旧的原则
一旦收到更大的新`epoch`的申请，马上让旧`epoch`的访问权失败，不再接受他们提交的取值。
然后给新`epoch`发放访问权，只接收新`epoch`提交的取值
- 新`epoch`可以抢占旧`epoch`，让旧`epoch`的访问权失败，旧`epoch`的`Proposer`将无法运行，新`epoch`的`Proposer`将开始运行
- 为了保持一致性，不同`epoch`的`Proposer`之间采用“后者认同前者”的原则
1、在肯定旧`epoch`无法生成确定性取值时，新的`epoch`会提交自己的`value`，不会冲突
2、旦旧`epoch`形成确定性取值，新的`epoch`可定可以获取到此值，并且会认同此取值，不会破坏
 
### 基于抢占式访问权的Acceptor的实现
- `Acceptor`保存的状态
当前`var`的取值`<accepted_epoch, accepted_value>`
最新发放访问权的`epoch(lasted_prepared_epoch)`
- `Acceptor::preprare(epoch)`:
只接收比`lasted_prepared_epoch`更大的`epoch`，并给予访问权，记录`lasted_prepares_epoch=epoch`
- `Acceptor::accept(var ,prepared_epoch, V )`:
验证`lasted_prepared_epoch == prepared_epoch` ,并设置`var`的取值`<accepted_epoch,accepted_value> = <prepared_epoch,v>`.
 
### Proposer(var , V)的两个阶段实现
- 第一阶段：获取`epoch` 轮次的访问权和当前var的取值
简单选取当前世界戳为`epoch`，通`Acceptor::prepare(epoch)`，获取`epoch`轮次的访问权和当前`var`的取值如果不能获取，返回`<error>`
- 第二阶段：采用“后者认同前者”的原则选定取值，进行提交。
1、如果`var`的取值为空，则可定当前没有确定性取值，则通过`Acceptor::accept(var, epoch , V)`提交数据`V`，成功后返回`<ok,V>`;如果`Acceptor`失败，返回`<error>`(被`epoch`抢占或者`Acceptor`故障)
2.如果`var`取值存在，则此值肯定是确定性取值，此时认同它不再更改，直接返回`<ok, epoch_old , accepted_value>`
 
> 总结

基于抢占式访问权的核心思想：
让`Proposer`将按照`epoch`递增的顺讯抢占式的依次运行，后者认同前者。可以避免`proposer`机器故障带来的死锁问题，并且可以`var`取值的一致性。但是还得引入多`Acceptor`单机模块`Acceptor`是故障导致整个系统`down`机 ，无法提供服务
 
- 关于方案1
如何控制`proposer`的并发运行？
    让`Propose`按照`epoch`递增的顺序抢占式的依次运行，后者会认同前者.(*注*：即使这样做了其实也不能够确保，因为在分布式中，获取的时钟值可能是一样的，如果真出现这种情况，`Acceptor`应该收回发出去的锁，让`Propose`重新发起申请，知道时钟是时钟是不一样的;还有就是抢占会发生`洪水`式的抢占失败，因为前者被后者抢占了，如果后者还没有执行第二阶段，那么前者也有重新抢占的可能，这样反复抢占（简称活锁），那就坑爹了)

为何可以保证一致性？
    `var`值只能被改动一次

为什么会有死锁问题？
    如果获取到的锁的`Propose`还有执行第二阶段，然后`down`机了，那么就会出现死锁的情况，但是如果使用抢占式的话还是可以避免到死锁的问题(如果让锁具有`时效性`，这样也是避免死锁的一种方案).
- 关于方案2
    如何解决方案1的死锁问题？
    在什么情况下，`Proposer`可以将`var`的取值确定为自己提供的取值？
 
## 确定一个不可变量的取值---方案2序
- 基于抢占访问权的核心思想
让`Propose`将按照`epoch`递增的顺序抢占式的依次运行，后者认同前者

- 可以避免`Proposer`机器故障带来的死锁问题，并且可以保证`var`取值的一致性

- 仍需要引入多`acceptor`
单机模块是故障导致整个系统`down`机，无法提供服务。

## 确定一个不可变量的取值---Paxos
- Paxos在方案2的基础上引入多`Acceptor`
`Acceptor`的实现保持不变，仍采用“喜新厌旧”的原则运行。
- Paxos采用“少数服从多数”的思路
- 一旦某`epoch`的取值`f`被半数`acceptor`接受，则认为此`var`取值被确定为`f`
,不在改变。

 
- Propose(var, v)第一阶段：选定某个`epoch`,获取epoch访问权和对应的`var`取值
获取半数以上`Acceptor`的访问权和对应的一组`var`取值

- Propose(var,v)第二阶段：采用“后者认同前者”的原则运行
1、肯定旧`epoch`无法生成稳定性取值时，新的`epoch`会提交自己的取值，不会冲突。
2、一旦旧`epoch`形成确定性取值，新的`epoch`肯定可以获取到此取值，并且会认同取值，不会破坏。
3、如果获取的`var`取值为空，则旧`epoch`无法形成确定性取值。此时努力使`<epoch, v>`成为确定性取值。
    - 向`epoch`对应的所有`acceptor`提交取值`<epoch, v>`.
    - 如果收到半数以上成功, 则返回`<ok, v>`
    - 否则，则返回`<error>`(被新`epoch`抢占或者`acceptor`故障)
4、如果`var`的取值存在，认同最大`acceptor_epoch`对应的取值`f`,努力使`<epoch, f>`成为确定性取值。

抢占式访问权机制的运行过程

<center>![](http://laohanlinux.github.io/images/img/blog/erlang/paxos/20140318102939343.png)
图一
![](http://laohanlinux.github.io/images/img/blog/erlang/paxos/20140318103033921.png)
图二（#1表示P1获取了访问权）
![](http://laohanlinux.github.io/images/img/blog/erlang/paxos/20140318103155250.png)
图3(现在是P2抢占获得了访问权，而P1保存着过期的访问权，此时Acceptor里面的lasted_prepared_epoch 比P1的大)
![](http://laohanlinux.github.io/images/img/blog/erlang/paxos/20140318103235265.png)
图4（P2成功设置了V2）
![](http://laohanlinux.github.io/images/img/blog/erlang/paxos/20140318103312546.png)
图5（P1的访问权已经失效了，所以Acceptor(#1,V1)失败）
![](http://laohanlinux.github.io/images/img/blog/erlang/paxos/20140318103359796.png)
图6（P3prepare(#3)成功，但是会返回<ok.#2,V2>，因为var已经被设置了，起到了一致性作用，所以不会在更改var的值）
</center>

## Paxos算法的核心思想

- 在抢占访问权的基础上引入多`Acceptor`
- 保证一个`epoch`,只有一个`propose`运行，`proposer`按照`epoch`递增的顺序依次执行
- 新`epoch`的`proposer`采用“后者认同前者”的思想运行
1、在肯旧`epoch`无法生成确定性取值时，新的`epoch`会提交自己的取值，不会冲突。
2、一旦旧`epoch`形成确定性取值，新的`epoch`肯定可以获取到此取值，并且认同此取值，不会破坏
- paxos算法可以满足容错要求
1、半数一下`acceptor`出现故障时，存活的`acceptor`仍然可以生成`var`的确定性取值。
2、一旦`var`取值被确定，即使出现半数以下`acceptor`故障，此取值可以被获取，并且将不再被更改。
- paxos算法的Liveness问题
新轮次的抢占会让旧轮次停止运行，如果每一轮次在第二阶成功之前都被新一轮抢占，则导致活锁，怎么解决呢？？？

可以参考`raft`选举的实现，可以使用随机时间戳来避免活锁，就说每个`Propse`每轮次的间隔时间是在某个时间段里的某个值，根据`raft`算法的验证，这种方案接近100%解决这种问题（忘记多少了，有兴趣的朋友，可以到raft的官网去查）。
