---
title: muduo net library chapter 1
date: 2015-07-01 00:04:49
tags:
- muduo 
categories:
- muduo 
- network program

description: muduo 是一个linux 网络库，使用poll和epoll模式。 这些内容是本人在大学的时候，通过  << c++教程网 >>  所学的知识笔记. 已经过去好几年了，版本也是比较老的了，属于入门级别的教程； 如有错误，纯属正常!!!

---


## [25] muduo_net库源码分析

### TCP网络编程本质

TCP网络编程最本质是的处理三个半事件

- 连接建立：服务器accept（被动）接受连接，客户端connect（主动）发起连接
- 连接断开：主动断开（close、shutdown），被动断开（read返回0）
- 消息到达：文件描述符可读
- 消息发送完毕：这算半个。对于低流量的服务，可不必关心这个事件;这里的发送完毕是指数据写入操作系统缓冲区，将由TCP协议栈负责数据的发送与重传，不代表对方已经接收到数据。



### EchoServer类图   

<center>![](http://laohanlinux.github.io/images/img/blog/muduo-net-library-chapter-1/EchoServer.png)</center>

*什么都不做的EventLoop*

- one loop   per thread

意思是说每个线程最多只能有一个EventLoop对象。

- EventLoop对象构造的时候，会检查当前线程是否已经创建了其他EventLoop对象，如果已创建，终止程序（LOG_FATAL）EventLoop构造函数会记住本对象所属线程（threadId_）。
- 创建了EventLoop对象的线程称为IO线程，其功能是运行事件循环（EventLoop::loop）

### EventLoop 头文件

EventLoop.h

``` c++
  // Copyright 2010, Shuo Chen.  All rights reserved.
  // http://code.google.com/p/muduo/
  //
  // Use of this source code is governed by a BSD-style license
  // that can be found in the License file.
  
  // Author: Shuo Chen (chenshuo at chenshuo dot com)
  //
  // This is a public header file, it must only include public header files.
  
  #ifndef MUDUO_NET_EVENTLOOP_H
  #define MUDUO_NET_EVENTLOOP_H
  
  #include <boost/noncopyable.hpp>
  
  #include <muduo/base/CurrentThread.h>
  #include <muduo/base/Thread.h>
  
  namespace muduo
  {
  namespace net
  {
  
  ///
  /// Reactor, at most one per thread.
  ///
  /// This is an interface class, so don't expose too much details.
  class EventLoop : boost::noncopyable
  {
  public:
    EventLoop();
    ~EventLoop();  // force out-line dtor, for scoped_ptr members.
  
    ///
    /// Loops forever.
    ///
    /// Must be called in the same thread as creation of the object.
    ///
    void loop();
  
    //断言是否是当前的线程
  
    void assertInLoopThread()
    {
        //过是当前线程，直接跳过，否则调用abortNotInLoopThread
      if (!isInLoopThread())
      {
        abortNotInLoopThread();
      }
    }
    //测试是否为当前线程
    bool isInLoopThread() const { return threadId_ == CurrentThread::tid(); }
  
    static EventLoop* getEventLoopOfCurrentThread();
  
  private:
    void abortNotInLoopThread();
  
    bool looping_; /* atomic , 是否处于事件循环*/
    const pid_t threadId_;        // 当前对象所属线程ID
  };
  
  }
  }
  #endif  // MUDUO_NET_EVENTLOOP_H
  ```

  ### EventLoop 源文件

  EventLoop.cpp

  ``` c++
  // Copyright 2010, Shuo Chen.  All rights reserved.
  // http://code.google.com/p/muduo/
  //
  // Use of this source code is governed by a BSD-style license
  // that can be found in the License file.
  
  // Author: Shuo Chen (chenshuo at chenshuo dot com)
  
  #include <muduo/net/EventLoop.h>
  
  #include <muduo/base/Logging.h>
  
  #include <poll.h>
  
  using namespace muduo;
  using namespace muduo::net;
  
  namespace
  {
  // 当前线程EventLoop对象指针
  // 线程局部存储 ，如果不用__thread修饰的话，那么就是线程共共享的了
  //这样达不到目标，。。
  //初始化时，设为0就OK了
  __thread EventLoop* t_loopInThisThread = 0;
  }
  
  EventLoop* EventLoop::getEventLoopOfCurrentThread()
  {
    return t_loopInThisThread;
  }
  
  EventLoop::EventLoop()
    : looping_(false), //初始化为false ，表示当前还没有处于事件循环状态
      threadId_(CurrentThread::tid()) //当前的线程Id ，用于标识
  {
    LOG_TRACE << "EventLoop created " << this << " in thread " << threadId_;
    // 如果当前线程已经创建了EventLoop对象，终止(LOG_FATAL),否者的话，设为当前线程this
    if (t_loopInThisThread)
    {
      LOG_FATAL << "Another EventLoop " << t_loopInThisThread
                << " exists in this thread " << threadId_;
    }
    else
    {
      t_loopInThisThread = this;
    }
  }
  
  EventLoop::~EventLoop()
  {
    t_loopInThisThread = NULL;
  }
  
  // 事件循环，该函数不能跨线程调用
  // 只能在创建该对象的线程中调用
  void EventLoop::loop()
  {
  //断言还没有事件循环
    assert(!looping_);
    // 断言当前处于创建该对象的线程中
    assertInLoopThread();
    //把事件循环标识设为true
    looping_ = true;
    LOG_TRACE << "EventLoop " << this << " start looping";
    //关注事件为NULL，个数为0，延时5000
    ::poll(NULL, 0, 5*1000);
  
    LOG_TRACE << "EventLoop " << this << " stop looping";
    //事件循环标识设为false ，这只是测试程序
    looping_ = false;
  }
  
  //终止程序
  void EventLoop::abortNotInLoopThread()
  {
    LOG_FATAL << "EventLoop::abortNotInLoopThread - EventLoop " << this
              << " was created in threadId_ = " << threadId_
              << ", current thread id = " <<  CurrentThread::tid();
  }
```

### EventLoop 的测试程序1

``` c++
#include <muduo/net/EventLoop.h>
 
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
/*
该程序 主要是用来测试EventLoop 是否是 一个Thread一个EventLoop
**/
void threadFunc()
{
    printf("threadFunc(): pid = %d, tid = %d\n",
        getpid(), CurrentThread::tid());
 
    EventLoop loop;
    loop.loop();
}
 
int main(void)
{
    printf("main(): pid = %d, tid = %d\n",
        getpid(), CurrentThread::tid());
 
    EventLoop loop;
 
    Thread t(threadFunc);
    t.start();
 
    loop.loop();
    t.join();
    return 0;
}
```
### 程序输出

``` c++
[root@localhost bin]# ./reactor_test01
main(): pid = 2724, tid = 2724
20131018 04:12:55.305234Z  2724 TRACE EventLoop EventLoop created 0xBFA6B2C0 in thread 2724 - EventLoop.cc:36
20131018 04:12:55.305599Z  2724 TRACE loop EventLoop 0xBFA6B2C0 start looping - EventLoop.cc:62
threadFunc(): pid = 2724, tid = 2725
20131018 04:12:55.305792Z  2725 TRACE EventLoop EventLoop created 0xB77E0068 in thread 2725 - EventLoop.cc:36
20131018 04:12:55.305809Z  2725 TRACE loop EventLoop 0xB77E0068 start looping - EventLoop.cc:62
20131018 04:13:00.321713Z  2724 TRACE loop EventLoop 0xBFA6B2C0 stop looping - EventLoop.cc:66
20131018 04:13:00.321779Z  2725 TRACE loop EventLoop 0xB77E0068 stop looping - EventLoop.cc:66
[root@localhost bin]#
```

### 测试程序2

``` c++
#include <muduo/net/EventLoop.h>
 
#include <stdio.h>
/**
该程序主要用来测试，多个线程使用同一个eventloop时，程序将会错误终止！！
**/
using namespace muduo;
using namespace muduo::net;
 
EventLoop* g_loop;
 
void threadFunc()
{
    g_loop->loop();
}
 
int main(void)
{
    EventLoop loop;
    g_loop = &loop;
    Thread t(threadFunc);
    t.start();
    t.join();
    return 0;
}
```

程序输出

```
[root@localhost bin]# ./reactor_test02
20131018 04:15:09.010234Z  2730 TRACE EventLoop EventLoop created 0xBFD53730 in thread 2730 - EventLoop.cc:36
20131018 04:15:09.010768Z  2731 FATAL EventLoop::abortNotInLoopThread - EventLoop 0xBFD53730 was created in threadId_ = 2730, current thread id = 2731 - EventLoop.cc:72
Aborted
[root@localhost bin]#
```

## [27] EpollPoller 

<center>![](http://laohanlinux.github.io/images/img/blog/muduo-net-library-chapter-1/Untitled.png)</center>

### EpollPoller的头文件

epollpller.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is an internal header file, you should not include this.
 
#ifndef MUDUO_NET_POLLER_EPOLLPOLLER_H
#define MUDUO_NET_POLLER_EPOLLPOLLER_H
 
#include <muduo/net/Poller.h>
 
#include <map>
#include <vector>
 
struct epoll_event;
 
namespace muduo
{
namespace net
{
 
///
/// IO Multiplexing with epoll(4).
///
class EPollPoller : public Poller
{
 public:
  EPollPoller(EventLoop* loop);
  virtual ~EPollPoller();
  // timeoutMs 超时事件
  // activeChannels活动通道
  virtual Timestamp poll(int timeoutMs, ChannelList* activeChannels);
  //更新通道
  virtual void updateChannel(Channel* channel);
  //移除通道
  virtual void removeChannel(Channel* channel);
 
 private:
  // EventListd的初始值
  static const int kInitEventListSize = 16;
 
  void fillActiveChannels(int numEvents,
                          ChannelList* activeChannels) const;
  //更新
  void update(int operation, Channel* channel);
 
  typedef std::vector<struct epoll_event> EventList;
 
  typedef std::map<int, Channel*> ChannelMap;
 
  //文件描述符 = epoll_create1(EPOLL_CLOEXEC)，用来表示要关注事件的fd的集合的描述符
  int epollfd_;
  // epoll_wait返回的活动的通道channelList
  EventList events_;
  //通道map
  ChannelMap channels_;
};
 
}
}
#endif  // MUDUO_NET_POLLER_EPOLLPOLLER_H
```

### EpollPoller的源文件

epollpoller.cc

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/net/poller/EPollPoller.h>
 
#include <muduo/base/Logging.h>
#include <muduo/net/Channel.h>
 
#include <boost/static_assert.hpp>
 
#include <assert.h>
#include <errno.h>
#include <poll.h>
#include <sys/epoll.h>
 
using namespace muduo;
using namespace muduo::net;
 
// On Linux, the constants of poll(2) and epoll(4)
// are expected to be the same.
BOOST_STATIC_ASSERT(EPOLLIN == POLLIN);
BOOST_STATIC_ASSERT(EPOLLPRI == POLLPRI);
BOOST_STATIC_ASSERT(EPOLLOUT == POLLOUT);
BOOST_STATIC_ASSERT(EPOLLRDHUP == POLLRDHUP);
BOOST_STATIC_ASSERT(EPOLLERR == POLLERR);
BOOST_STATIC_ASSERT(EPOLLHUP == POLLHUP);
 
namespace
{
const int kNew = -1; //新通道
const int kAdded = 1; //要关注的通道
const int kDeleted = 2; //要删除的通道
}
 
EPollPoller::EPollPoller(EventLoop* loop)
  : Poller(loop),
    epollfd_(::epoll_create1(EPOLL_CLOEXEC)),
    events_(kInitEventListSize)
{
  if (epollfd_ < 0)
  {
    LOG_SYSFATAL << "EPollPoller::EPollPoller";
  }
}
 
EPollPoller::~EPollPoller()
{
  ::close(epollfd_);
}
 
// IO 线程
Timestamp EPollPoller::poll(int timeoutMs, ChannelList* activeChannels)
{
  //监听事件的到来
  int numEvents = ::epoll_wait(epollfd_,
                               &*events_.begin(),
                               static_cast<int>(events_.size()),
                               timeoutMs);
 
  Timestamp now(Timestamp::now());
  if (numEvents > 0)
  {
    LOG_TRACE << numEvents << " events happended";
    /*添加活动通道channel*/
    fillActiveChannels(numEvents, activeChannels);
    //如果活动通道的容器已满，则增加活动通道容器的容量
    if (implicit_cast<size_t>(numEvents) == events_.size())
    {
      events_.resize(events_.size()*2);
    }
  }
  else if (numEvents == 0)
  {
    LOG_TRACE << " nothing happended";
  }
  else
  {
    LOG_SYSERR << "EPollPoller::poll()";
  }
  return now;
}
 
/*添加活动通道channel*/
void EPollPoller::fillActiveChannels(int numEvents,
                                     ChannelList* activeChannels) const
{
  assert(implicit_cast<size_t>(numEvents) <= events_.size());
  for (int i = 0; i < numEvents; ++i)
  {
    Channel* channel = static_cast<Channel*>(events_[i].data.ptr);
    //如果是调试状态，则
#ifndef NDEBUG
    int fd = channel->fd();
    ChannelMap::const_iterator it = channels_.find(fd);
    assert(it != channels_.end());
    assert(it->second == channel);
#endif
    //否则直接跳到这里
    // 设置channel的“可用事件”
    channel->set_revents(events_[i].events);
    // 加入活动通道容器
    activeChannels->push_back(channel);
  }
}
 
//更新某个通道 channel
void EPollPoller::updateChannel(Channel* channel)
{
  //断言 在IO线程中
  Poller::assertInLoopThread();
  LOG_TRACE << "fd = " << channel->fd() << " events = " << channel->events();
 
  //channel 的默认值是 -1 ， ---->>channel class
  const int index = channel->index();
  // kNew 表示有新的通道要增加，kDeleted表示将已不关注事件的fd重新关注事件，及时重新加到epollfd_中去
  if (index == kNew || index == kDeleted)
  {
    // a new one, add with EPOLL_CTL_ADD
    int fd = channel->fd();
    if (index == kNew)
    {
      //如果是新的channel，那么在channels_里面是找不到的
      assert(channels_.find(fd) == channels_.end());
      //添加到channels_中
      channels_[fd] = channel;
    }
 
    else // index == kDeleted
    {
      assert(channels_.find(fd) != channels_.end());
      assert(channels_[fd] == channel);
    }
    //
    channel->set_index(kAdded);
    update(EPOLL_CTL_ADD, channel);
  }
  else
  {
    // update existing one with EPOLL_CTL_MOD/DEL
    int fd = channel->fd();
    (void)fd;
    assert(channels_.find(fd) != channels_.end());
    assert(channels_[fd] == channel);
    //断言已经在channels_里面了，并且已在epollfd_中
    assert(index == kAdded);
 
    //剔除channel的关注事件
    //如果channel没有事件关注了，就把他从epollfd_中剔除掉
    if (channel->isNoneEvent())
    {
      update(EPOLL_CTL_DEL, channel);
      //更新index = kDeleted
      channel->set_index(kDeleted);
    }
    else
    {
      update(EPOLL_CTL_MOD, channel);
    }
  }
}
 
// 移除channel
void EPollPoller::removeChannel(Channel* channel)
{
  //断言实在IO线程中
  Poller::assertInLoopThread();
  int fd = channel->fd();
  LOG_TRACE << "fd = " << fd;
  //断言能在channels_里面找到channel
  assert(channels_.find(fd) != channels_.end());
  assert(channels_[fd] == channel);
  //断言所要移除的channel已经没有事件关注了，但是此时在event_里面可能还有他的记录
  assert(channel->isNoneEvent());
  int index = channel->index();
  //断言
  assert(index == kAdded || index == kDeleted);
  //真正从channels_里面删除掉channel
  size_t n = channels_.erase(fd);
  (void)n;
  assert(n == 1);
 
  //从event_中剔除channel
  if (index == kAdded)
  {
    update(EPOLL_CTL_DEL, channel);
  }
 
  // channel现在变成新的channel了
  channel->set_index(kNew);
}
 
void EPollPoller::update(int operation, Channel* channel)
{
  struct epoll_event event;
  bzero(&event, sizeof event);
  event.events = channel->events();
  event.data.ptr = channel;
  int fd = channel->fd();
  //更新操作
  if (::epoll_ctl(epollfd_, operation, fd, &event) < 0)
  {
    //写入日志
    if (operation == EPOLL_CTL_DEL)
    {
      LOG_SYSERR << "epoll_ctl op=" << operation << " fd=" << fd;
    }
    else
    {
      LOG_SYSFATAL << "epoll_ctl op=" << operation << " fd=" << fd;
    }
  }
}
```

### 测试程序

Reactor_test03.cc

``` c++
#include <muduo/net/Channel.h>
#include <muduo/net/EventLoop.h>
 
#include <boost/bind.hpp>
 
#include <stdio.h>
#include <sys/timerfd.h>
 
using namespace muduo;
using namespace muduo::net;
 
EventLoop* g_loop;
int timerfd;
 
void timeout(Timestamp receiveTime)
{
    printf("Timeout!\n");
    uint64_t howmany;
    ::read(timerfd, &howmany, sizeof howmany);
    g_loop->quit();
}
 
int main(void)
{
    EventLoop loop;
    g_loop = &loop;
 
    timerfd = ::timerfd_create(CLOCK_MONOTONIC, TFD_NONBLOCK | TFD_CLOEXEC);
    Channel channel(&loop, timerfd);
    channel.setReadCallback(boost::bind(timeout, _1));
    channel.enableReading();
 
    struct itimerspec howlong;
    bzero(&howlong, sizeof howlong);
    howlong.it_value.tv_sec = 1;
    ::timerfd_settime(timerfd, 0, &howlong, NULL);
 
    loop.loop();
 
    ::close(timerfd);
}
```

程序输出

``` sh
[root@localhost bin]# ./reactor_test03
20131020 02:24:05.657327Z  4009 TRACE EventLoop EventLoop created 0xBFD2AAD4 in thread 4009 - EventLoop.cc:42
20131020 02:24:05.657513Z  4009 TRACE updateChannel fd = 4 events = 3 - EPollPoller.cc:104
20131020 02:24:05.657554Z  4009 TRACE loop EventLoop 0xBFD2AAD4 start looping - EventLoop.cc:68
20131020 02:24:06.658756Z  4009 TRACE poll 1 events happended - EPollPoller.cc:65
20131020 02:24:06.658972Z  4009 TRACE printActiveChannels {4: IN }  - EventLoop.cc:139
Timeout!
20131020 02:24:06.659008Z  4009 TRACE loop EventLoop 0xBFD2AAD4 stop looping - EventLoop.cc:93
[root@localhost bin]#
```

## [28] muduo的定时器由三个类实现，TimerId（定时器）、Timer(最上层的抽象)、TimerQueue（定时器的列表）

- muduo的定时器由三个类实现，TimerId（定时器）、Timer(最上层的抽象,并没有调用定时器的相关函数)、TimerQueue（定时器的列表），用户只能看到第一个类，其它两个都是内部实现细节
- TimerQueue的接口很简单，只有两个函数addTimer和cancel。实际上这两个函数也没有向外部开发，访问它要通过EventLoop的相关方法
- EventLoop
```
       runAt    在某个时刻运行定时器------>TimerQueue.addTimer

       runAfter    过一段时间运行定时器---->TimeQueue.addTimer

       runEvery    每隔一段时间运行定时器->TimerQueue.addTimer

       cancel    取消定时器 ----》TimeQueue.cancal
```
- TimerQueue数据结构的选择，能快速根据当前时间找到已到期的定时器，也要高效的添加和删除Timer，因而可以用二叉搜索树，用map或者set;如果选择map的话，是有问题的，可能TimeQueue里面有时间戳是一样的，但定时器timer*是不一样的，可以考虑multimap，不过最后还是不要使用

``` c++
 typedef std::pair<Timestamp, Timer*> Entry;
 typedef std::set<Entry> TimerList;
```

<center>![](http://laohanlinux.github.io/images/img/blog/muduo-net-library-chapter-1/%E5%AE%9A%E6%97%B6%E5%99%A828-2.png)</center>

### 时序图

<center>![](http://laohanlinux.github.io/images/img/blog/muduo-net-library-chapter-1/%E5%AE%9A%E6%97%B6%E5%99%A828-1.png)</center>

### Timer头文件

timer.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is an internal header file, you should not include this.
 
#ifndef MUDUO_NET_TIMER_H
#define MUDUO_NET_TIMER_H
 
#include <boost/noncopyable.hpp>
 
#include <muduo/base/Atomic.h>
#include <muduo/base/Timestamp.h>
#include <muduo/net/Callbacks.h>
 
namespace muduo
{
namespace net
{
///
/// Internal class for timer event.
///
class Timer : boost::noncopyable
{
 public:
  Timer(const TimerCallback& cb, Timestamp when, double interval)
    : callback_(cb),
      expiration_(when),
      interval_(interval),
      repeat_(interval > 0.0),
      sequence_(s_numCreated_.incrementAndGet())
  { }
 
  void run() const
  {
    callback_();
  }
 
  Timestamp expiration() const  { return expiration_; }
  bool repeat() const { return repeat_; }
  int64_t sequence() const { return sequence_; }
 
  void restart(Timestamp now);
 
  static int64_t numCreated() { return s_numCreated_.get(); }
 
 private:
  const TimerCallback callback_;        // 定时器回调函数
  Timestamp expiration_;                // 下一次的超时时刻
  const double interval_;               // 超时时间间隔，如果是一次性定时器，该值为0
  const bool repeat_;                   // 是否重复
  const int64_t sequence_;              // 定时器序号
 
  static AtomicInt64 s_numCreated_;     // 定时器计数，当前已经创建的定时器数量
};
}
}
#endif  // MUDUO_NET_TIMER_H
```

### Timer源文件

timer.cc

``` c++

// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/net/Timer.h>
 
using namespace muduo;
using namespace muduo::net;
 
AtomicInt64 Timer::s_numCreated_;
 
void Timer::restart(Timestamp now)
{
  if (repeat_)
  {
    // 重新计算下一个超时时刻
    expiration_ = addTime(now, interval_);
  }
  else
  {
    expiration_ = Timestamp::invalid();
  }
}
```

### TimerId头文件

timerid.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_TIMERID_H
#define MUDUO_NET_TIMERID_H
 
#include <muduo/base/copyable.h>
 
namespace muduo
{
namespace net
{
 
class Timer;
 
///是一个不透明的ID ，是外部可见的一个类
/// An opaque identifier, for canceling Timer.
///
class TimerId : public muduo::copyable
{
 public:
  TimerId()
    : timer_(NULL),
      sequence_(0)
  {
  }
 
  TimerId(Timer* timer, int64_t seq)
    : timer_(timer),
      sequence_(seq)
  {
  }
 
  // default copy-ctor, dtor and assignment are okay
 
  friend class TimerQueue;
 
 private:
  Timer* timer_; //定时器的地址 ，timer里面也包含了timer的序号
  int64_t sequence_; //定时器的序号
};
 
}
}
 
#endif  // MUDUO_NET_TIMERID_H
```

### TimerQueue头文件

timerqueue.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is an internal header file, you should not include this.
 
#ifndef MUDUO_NET_TIMER_H
#define MUDUO_NET_TIMER_H
 
#include <boost/noncopyable.hpp>
 
#include <muduo/base/Atomic.h>
#include <muduo/base/Timestamp.h>
#include <muduo/net/Callbacks.h>
 
namespace muduo
{
namespace net
{
///
/// Internal class for timer event.
///
class Timer : boost::noncopyable
{
 public:
  Timer(const TimerCallback& cb, Timestamp when, double interval)
    : callback_(cb),
      expiration_(when),
      interval_(interval),
      repeat_(interval > 0.0),
      sequence_(s_numCreated_.incrementAndGet())
  { }
 
  void run() const
  {
    callback_();
  }
 
  Timestamp expiration() const  { return expiration_; }
  bool repeat() const { return repeat_; }
  int64_t sequence() const { return sequence_; }
 
  void restart(Timestamp now);
 
  static int64_t numCreated() { return s_numCreated_.get(); }
 
 private:
  const TimerCallback callback_;        // 定时器回调函数
  Timestamp expiration_;                // 下一次的超时时刻
  const double interval_;               // 超时时间间隔，如果是一次性定时器，该值为0
  const bool repeat_;                   // 是否重复
  const int64_t sequence_;              // 定时器序号
 
  static AtomicInt64 s_numCreated_;     // 定时器计数，当前已经创建的定时器数量
};
}
}
#endif  // MUDUO_NET_TIMER_H
```

### TimerQueue源文件

timerqueue.cc

``` rust
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#define __STDC_LIMIT_MACROS
#include <muduo/net/TimerQueue.h>
 
#include <muduo/base/Logging.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/Timer.h>
#include <muduo/net/TimerId.h>
 
#include <boost/bind.hpp>
 
#include <sys/timerfd.h>
 
namespace muduo
{
namespace net
{
namespace detail
{
 
// 创建定时器
int createTimerfd()
{
  int timerfd = ::timerfd_create(CLOCK_MONOTONIC,
                                 TFD_NONBLOCK | TFD_CLOEXEC);
  if (timerfd < 0)
  {
    LOG_SYSFATAL << "Failed in timerfd_create";
  }
  return timerfd;
}
 
// 计算超时时刻与当前时间的时间差
struct timespec howMuchTimeFromNow(Timestamp when)
{
  int64_t microseconds = when.microSecondsSinceEpoch()
                         - Timestamp::now().microSecondsSinceEpoch();
  if (microseconds < 100)
  {
    microseconds = 100;
  }
  struct timespec ts;
  ts.tv_sec = static_cast<time_t>(
      microseconds / Timestamp::kMicroSecondsPerSecond);
  ts.tv_nsec = static_cast<long>(
      (microseconds % Timestamp::kMicroSecondsPerSecond) * 1000);
  return ts;
}
 
// 清除定时器，避免一直触发
void readTimerfd(int timerfd, Timestamp now)
{
  uint64_t howmany;
  ssize_t n = ::read(timerfd, &howmany, sizeof howmany);
  LOG_TRACE << "TimerQueue::handleRead() " << howmany << " at " << now.toString();
  if (n != sizeof howmany)
  {
    LOG_ERROR << "TimerQueue::handleRead() reads " << n << " bytes instead of 8";
  }
}
 
// 重置定时器的超时时间
void resetTimerfd(int timerfd, Timestamp expiration)
{
  // wake up loop by timerfd_settime()
  struct itimerspec newValue;
  struct itimerspec oldValue;
  bzero(&newValue, sizeof newValue);
  bzero(&oldValue, sizeof oldValue);
  newValue.it_value = howMuchTimeFromNow(expiration);
  int ret = ::timerfd_settime(timerfd, 0, &newValue, &oldValue);
  if (ret)
  {
    LOG_SYSERR << "timerfd_settime()";
  }
}
 
}
}
}
 
using namespace muduo;
using namespace muduo::net;
using namespace muduo::net::detail;
 
TimerQueue::TimerQueue(EventLoop* loop)
  : loop_(loop),
    timerfd_(createTimerfd()),
    timerfdChannel_(loop, timerfd_),
    timers_(),
    callingExpiredTimers_(false)
{
  timerfdChannel_.setReadCallback(
      boost::bind(&TimerQueue::handleRead, this));
  // we are always reading the timerfd, we disarm it with timerfd_settime.
  /*把timerfd 加入到poller来关注*/
  timerfdChannel_.enableReading();
}
 
TimerQueue::~TimerQueue()
{
  ::close(timerfd_);
  // do not remove channel, since we're in EventLoop::dtor();
  for (TimerList::iterator it = timers_.begin();
      it != timers_.end(); ++it)
  {
    delete it->second;
  }
}
/*增加一个定时器addTimer()----->addTimerInLoop()*/
TimerId TimerQueue::addTimer(const TimerCallback& cb,
                             Timestamp when,
                             double interval)
{
 
  /**
注意：addTimer虽然是线程安全的，但是这里把安全实现的代码给注释掉了，所以以下的代码必须在所属
的EventLoop IO线程中调用
  */
  Timer* timer = new Timer(cb, when, interval);
  /*
  loop_->runInLoop(
      boost::bind(&TimerQueue::addTimerInLoop, this, timer));
      */
  addTimerInLoop(timer);
  return TimerId(timer, timer->sequence());
}
 
void TimerQueue::cancel(TimerId timerId)
{
  /*
  loop_->runInLoop(
      boost::bind(&TimerQueue::cancelInLoop, this, timerId));
      */
  cancelInLoop(timerId);
}
/*添加定时器到所属eventloop IO线程中*/
void TimerQueue::addTimerInLoop(Timer* timer)
{
  loop_->assertInLoopThread();
  // 插入一个定时器，有可能会使得最早到期的定时器发生改变
  bool earliestChanged = insert(timer);
 
  if (earliestChanged)
  {
    // 重置定时器的超时时刻(timerfd_settime)
    resetTimerfd(timerfd_, timer->expiration());
  }
}
 
/*取消某个timer*/
void TimerQueue::cancelInLoop(TimerId timerId)
{
  loop_->assertInLoopThread();
  assert(timers_.size() == activeTimers_.size());
  ActiveTimer timer(timerId.timer_, timerId.sequence_);
  // 查找该定时器
  ActiveTimerSet::iterator it = activeTimers_.find(timer);
  if (it != activeTimers_.end())
  {
    size_t n = timers_.erase(Entry(it->first->expiration(), it->first));
    assert(n == 1); (void)n;
    delete it->first; // FIXME: no delete please,如果用了unique_ptr,这里就不需要手动删除了
    activeTimers_.erase(it);
  }
  else if (callingExpiredTimers_)
  {
    // 已经到期，并且正在调用回调函数的定时器 ， 那么将其加到cancelingtimers中，
    // 以便在其回调处理完后， 在reset(expired, now)里时无效，不需要重置
    cancelingTimers_.insert(timer);
  }
  assert(timers_.size() == activeTimers_.size());
}
 
/*定时器触发的回调函数*/
void TimerQueue::handleRead()
{
  loop_->assertInLoopThread();
  Timestamp now(Timestamp::now());
  readTimerfd(timerfd_, now);       // 清除该事件，避免一直触发
 
  // 获取该时刻之前所有的定时器列表(即超时定时器列表)
  std::vector<Entry> expired = getExpired(now);
  //已经处于处理到期定时器当中
  callingExpiredTimers_ = true;
  //清除上一次的超时定时器队列
  cancelingTimers_.clear();
  // safe to callback outside critical section
  for (std::vector<Entry>::iterator it = expired.begin();
      it != expired.end(); ++it)
  {
    // 这里回调定时器处理函数
    it->second->run();
  }
  callingExpiredTimers_ = false;
 
  // 不是一次性定时器，需要重启
  reset(expired, now);
}
 
// rvo 优化
//返回已经超时的timers
// TimerQueue = 1,1,1,3,4,5,7,9
// 那么触发第一个1时，其实后面的2个1也被触发了
// timerQueue 只管队列中的第一个定时器
std::vector<TimerQueue::Entry> TimerQueue::getExpired(Timestamp now)
{
  assert(timers_.size() == activeTimers_.size());
  std::vector<Entry> expired;
  Entry sentry(now, reinterpret_cast<Timer*>(UINTPTR_MAX));
  // 返回第一个未到期的Timer的迭代器
  // lower_bound的含义是返回第一个值>=sentry的元素的iterator
  // 即*end >= sentry，从而end->first > now
  TimerList::iterator end = timers_.lower_bound(sentry);
  assert(end == timers_.end() || now < end->first);
  // 将到期的定时器插入到expired中
  std::copy(timers_.begin(), end, back_inserter(expired));
  // 从timers_中移除到期的定时器
  timers_.erase(timers_.begin(), end);
 
  // 从activeTimers_中移除到期的定时器
  for (std::vector<Entry>::iterator it = expired.begin();it != expired.end(); ++it)
  {
    ActiveTimer timer(it->second, it->second->sequence());
    size_t n = activeTimers_.erase(timer);
    assert(n == 1); (void)n;
  }
 
  assert(timers_.size() == activeTimers_.size());
  return expired;
}
 
void TimerQueue::reset(const std::vector<Entry>& expired, Timestamp now)
{
  Timestamp nextExpire;
 
  for (std::vector<Entry>::const_iterator it = expired.begin();
      it != expired.end(); ++it)
  {
    ActiveTimer timer(it->second, it->second->sequence());
    // 如果是重复的定时器并且是未取消定时器（未被其他线程取消掉），则重启该定时器
    // cancelingTimers_.find(timer) == cancelingTimers_.end()  在取消的队列中找到不到timer
    if (it->second->repeat()&& cancelingTimers_.find(timer) == cancelingTimers_.end())
    {
      it->second->restart(now);
      insert(it->second);
    }
    else
    {
      // 一次性定时器或者已被取消的定时器是不能重置的，因此删除该定时器
      // FIXME move to a free list
      delete it->second; // FIXME: no delete please
    }
  }
 
  if (!timers_.empty())
  {
    // 获取最早到期的定时器超时时间
    nextExpire = timers_.begin()->second->expiration();
  }
 
  if (nextExpire.valid())
  {
    // 重置定时器的超时时刻(timerfd_settime)
    resetTimerfd(timerfd_, nextExpire);
  }
}
 
bool TimerQueue::insert(Timer* timer)
{
  loop_->assertInLoopThread();
  assert(timers_.size() == activeTimers_.size());
  // 最早到期时间是否改变
  bool earliestChanged = false;
  //获取传进来的定时器的超时时间
  Timestamp when = timer->expiration();
  TimerList::iterator it = timers_.begin();
  // 如果timers_为空或者when小于timers_中的最早到期时间
  if (it == timers_.end() || when < it->first)
  {
 
    earliestChanged = true;
  }
  {
    // 插入到timers_中
    std::pair<TimerList::iterator, bool> result
      = timers_.insert(Entry(when, timer));
    /*断言插入操作是否成功*/ 
    assert(result.second); (void)result;
  }
  {
    // 插入到activeTimers_中
    std::pair<ActiveTimerSet::iterator, bool> result
      = activeTimers_.insert(ActiveTimer(timer, timer->sequence()));
    assert(result.second); (void)result;
  }
  //断言是否成功操作了
  assert(timers_.size() == activeTimers_.size());
  return earliestChanged;
}
```

## [29] EventLoop再分析之IO线程（29）

### EventLoop IO 线程的简单描述

<center>![](http://laohanlinux.github.io/images/img/blog/muduo-net-library-chapter-1/eventloop.runInloop.png)</center>

eventloopThread 分析：

eventloop现成循环调用loop()函数：

- 获取活跃的通道
- 执行活跃通道的回调函数
- 调用`doPending Functors`，获取`queueLoop`的任务，依次执行(这些任务会阻塞IO线程，即IO线程也做了计算任务，所以这些任务不应该是耗时的任务)

> 注意
>
> eventFd在这里用于唤醒eventloop的事件，试想一下，加入当前只有IO线程的计算任务，那么eventloop会阻塞在poll()函数里面，直到超时，这是不可取的策略。

### 进程(线程)wait/notify

- pipe
- socketpair
- eventfd

`eventfd` 是一个比`pipe`更高效的线程间事件通知机制，一方面它比`pipe`少用一个`file descripor`，节省了资源；另一方面,`eventfd`的缓冲区管理也简单得多，全部`buffer`只有定长`8` `bytes`，不像`pipe`那样可能有不定长的真正`buffer`。

<center>![](http://laohanlinux.github.io/images/img/blog/muduo-net-library-chapter-1/eventloop.runInloop1.png)</center>
<center>![](http://laohanlinux.github.io/images/img/blog/muduo-net-library-chapter-1/eventloop.runInloop2.png)</center>

### EventLoop头文件

eventloop.h 

> NOTIFY: 
>
> 这里的`eventloop` 和 *muduo_net库源码分析（26-1)* 里面的内容有些不一样，这里的`eventloop`加了一些内容, 在阅读`eventloop`时，要注意`runInLoop`和`queueInloop`这两个函数，`wakeup（）`也得注意.

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_EVENTLOOP_H
#define MUDUO_NET_EVENTLOOP_H
 
#include <vector>
 
#include <boost/noncopyable.hpp>
#include <boost/scoped_ptr.hpp>
 
#include <muduo/base/Mutex.h>
#include <muduo/base/Thread.h>
#include <muduo/base/Timestamp.h>
#include <muduo/net/Callbacks.h>
#include <muduo/net/TimerId.h>
 
namespace muduo
{
namespace net
{
 
class Channel;
class Poller;
class TimerQueue;
///
/// Reactor, at most one per thread.
///
/// This is an interface class, so don't expose too much details.
class EventLoop : boost::noncopyable
{
 public:
  /* IO事件回调函数，就像“监听socketfd” ，我们可以把 监听socketfd 的处理函数 做成一个回调函数，
  把这个函数放到pendingFunctors_中，然后由IO线程自己回调处理*/
  typedef boost::function<void()> Functor;
 
  EventLoop();
  ~EventLoop();  // force out-line dtor, for scoped_ptr members.
 
  ///
  /// Loops forever.
  ///
  /// Must be called in the same thread as creation of the object.
  ///
  void loop();
 
  void quit();
 
  ///
  /// Time when poll returns, usually means data arrivial.
  ///
  Timestamp pollReturnTime() const { return pollReturnTime_; }
 
  /// Runs callback immediately in the loop thread.
  /// It wakes up the loop, and run the cb.
  /// If in the same loop thread, cb is run within the function.
  /// Safe to call from other threads.
  void runInLoop(const Functor& cb);
  /// Queues callback in the loop thread.
  /// Runs after finish pooling.
  /// Safe to call from other threads.
  void queueInLoop(const Functor& cb);
 
  // timers
 
  ///
  /// Runs callback at 'time'.
  /// Safe to call from other threads.
  ///
  TimerId runAt(const Timestamp& time, const TimerCallback& cb);
  ///
  /// Runs callback after @c delay seconds.
  /// Safe to call from other threads.
  ///
  TimerId runAfter(double delay, const TimerCallback& cb);
  ///
  /// Runs callback every @c interval seconds.
  /// Safe to call from other threads.
  ///
  TimerId runEvery(double interval, const TimerCallback& cb);
  ///
  /// Cancels the timer.
  /// Safe to call from other threads.
  ///
  void cancel(TimerId timerId);
 
  // internal usage
  void wakeup();
  void updateChannel(Channel* channel);     // 在Poller中添加或者更新通道
  void removeChannel(Channel* channel);     // 从Poller中移除通道
 
  void assertInLoopThread()
  {
    if (!isInLoopThread())
    {
      abortNotInLoopThread();
    }
  }
  bool isInLoopThread() const { return threadId_ == CurrentThread::tid(); }
 
  bool eventHandling() const { return eventHandling_; }
 
  static EventLoop* getEventLoopOfCurrentThread();
 
 private:
  void abortNotInLoopThread();
  void handleRead();  // waked up
  void doPendingFunctors();
 
  void printActiveChannels() const; // DEBUG
 
  typedef std::vector<Channel*> ChannelList;
 
  bool looping_; /* atomic */
  bool quit_; /* atomic */
  bool eventHandling_; /*事件处理函数状态 atomic */
  bool callingPendingFunctors_; /*是否处于IO回调函数的处理状态中 atomic */
  const pid_t threadId_;        // 当前对象所属线程ID
  Timestamp pollReturnTime_;
  boost::scoped_ptr<Poller> poller_;
  boost::scoped_ptr<TimerQueue> timerQueue_;
 
/*****
 
wakeupFd_ : 这个描述符主要是未了
 
****/
  int wakeupFd_;                // 用于eventfd所创建的文件描述符
  // unlike in TimerQueue, which is an internal class,
  // we don't expose Channel to client.
  /*
  wakeupChannel 和EventLoop 是组合的关系，生命周期由EventLoop管理
  一定要看下面的内容，如果不明白下面的内容的话，代码是很难懂的（个人观点，高手勿喷！！^V^）
  当其他线程跟IO线程通信时， 他们使用的是同一个通道(wakeupChannel_) ， 当其他线程wakeup（write）时，
poll会返回。然后IO执行handleRead，但是主要的回调函数不是handleRead ，而是pendingFunctors_里面的函数，
hadnleRead只是为了不被多次触发而已，也就是说其他线程要想IO线程执行他们所需的函调函数，只要把回调函数
注入到pendingFunctors_，然后wakeup（write）唤醒一下。
  当然IO线程也可以和IO线程进行通信，但是情况有些不一样，具体哪些不同，大家自己看着办吧！
 
  注： IO线程就是EventLoop所属的线程
  */
  boost::scoped_ptr<Channel> wakeupChannel_;  //wakeupFd_所对应的通道 该通道将会纳入poller_来管理
  ChannelList activeChannels_;      // Poller返回的活动通道
  Channel* currentActiveChannel_;   // 当前正在处理的活动通道
  MutexLock mutex_;
  // IO线程 回调函数容器
  std::vector<Functor> pendingFunctors_; // @BuardedBy mutex_
};
 
}
}
#endif  // MUDUO_NET_EVENTLOOP_H
```

### EventLoop源文件

这里的`eventloop`和 *muduo_net库源码分析（26-1）*里面的内容有些不一样，这里的`eventloop`加了一些内容，具体情况可以回去看一看.

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/net/EventLoop.h>
 
#include <muduo/base/Logging.h>
#include <muduo/net/Channel.h>
#include <muduo/net/Poller.h>
#include <muduo/net/TimerQueue.h>
 
//#include <poll.h>
#include <boost/bind.hpp>
 
#include <sys/eventfd.h>
 
using namespace muduo;
using namespace muduo::net;
 
namespace
{
// 当前线程EventLoop对象指针
// 线程局部存储
__thread EventLoop* t_loopInThisThread = 0;
 
const int kPollTimeMs = 10000;
 
int createEventfd()
{
  int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
  if (evtfd < 0)
  {
    LOG_SYSERR << "Failed in eventfd";
    abort();
  }
  return evtfd;
}
 
}
 
EventLoop* EventLoop::getEventLoopOfCurrentThread()
{
  return t_loopInThisThread;
}
 
EventLoop::EventLoop()
  : looping_(false),
    quit_(false),
    eventHandling_(false),
    callingPendingFunctors_(false),
    threadId_(CurrentThread::tid()),
    poller_(Poller::newDefaultPoller(this)),
    timerQueue_(new TimerQueue(this)),
    wakeupFd_(createEventfd()), //创建 wakeupFd_
    wakeupChannel_(new Channel(this, wakeupFd_)), //创建wakeupFd_对应的channel
    currentActiveChannel_(NULL)
{
  LOG_TRACE << "EventLoop created " << this << " in thread " << threadId_;
  // 如果当前线程已经创建了EventLoop对象，终止(LOG_FATAL)
  if (t_loopInThisThread)
  {
    LOG_FATAL << "Another EventLoop " << t_loopInThisThread
              << " exists in this thread " << threadId_;
  }
  else
  {
    t_loopInThisThread = this;
  }
 
  // 设定wakeupChannel的回调函数
  wakeupChannel_->setReadCallback(
      boost::bind(&EventLoop::handleRead, this));
  // we are always reading the wakeupfd
  //纳入poller来管理
  wakeupChannel_->enableReading();
}
 
EventLoop::~EventLoop()
{
  ::close(wakeupFd_);
  t_loopInThisThread = NULL;
}
 
// 事件循环，该函数不能跨线程调用
// 只能在创建该对象的线程中调用
void EventLoop::loop()
{
  assert(!looping_);
  // 断言当前处于创建该对象的线程中
  assertInLoopThread();
  looping_ = true;
  quit_ = false;
  LOG_TRACE << "EventLoop " << this << " start looping";
 
  //::poll(NULL, 0, 5*1000);
  while (!quit_)
  {
    activeChannels_.clear();
    pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);
    //++iteration_;
    if (Logger::logLevel() <= Logger::TRACE)
    {
      printActiveChannels();
    }
    // TODO sort channel by priority
    eventHandling_ = true;
    for (ChannelList::iterator it = activeChannels_.begin();
        it != activeChannels_.end(); ++it)
    {
      currentActiveChannel_ = *it;
      currentActiveChannel_->handleEvent(pollReturnTime_);
    }
    currentActiveChannel_ = NULL;
    eventHandling_ = false;
    // 让IO线程也能执行一些计算任务
    doPendingFunctors();
  }
 
  LOG_TRACE << "EventLoop " << this << " stop looping";
  looping_ = false;
}
 
// 该函数可以跨线程调用
void EventLoop::quit()
{
  quit_ = true;
  //如果不是由当前IO线程调用的，那么我们还要唤醒IO线程
  if (!isInLoopThread())
  {
    wakeup();
  }
}
/*
sublime 注释的好方法，都是苦力活
===========         ===================               ======         ===================
||        |||      |||=================             =                         {}
||        ****      ||                             =                          {}
||       ****       ||                            =                           {}
|||||||||           ||=================              =  = = =                 {}
||      \\\         ||==================                      ///             {}
||        \\        ||                                      = //              {}
                    //                                        //              {}
||        \\\       || ==================                    //               {}
||          \\      ||===================           =========                 {}
*/
// 在I/O线程中注册某个回调函数，该函数可以跨线程调用
void EventLoop::runInLoop(const Functor& cb)
{
  if (isInLoopThread())
  {
    // 如果是当前IO线程调用runInLoop，则同步调用cb
    cb();
  }
  else
  {
    // 如果是其它线程调用runInLoop，则异步地将cb添加到队列，让EventLoop线程来执行这个cb（）函数
    queueInLoop(cb);
  }
}
 
// 把IO回调函数注册到pendingFunctors_中
void EventLoop::queueInLoop(const Functor& cb)
{
  {
    // 保护临界区
  MutexLockGuard lock(mutex_);
    // 将任务添加到任务队列中
  pendingFunctors_.push_back(cb);
  }
 
  // 1.--->调用queueInLoop的线程不是IO线程需要唤醒，以便IO线程及时处理任务
  // 2.--->或者调用queueInLoop的线程是IO线程，并且此时正在调用pending functor，需要唤醒
   /*   如果我们不唤醒，那么下一次poll就没法获取该cb的触发事件，这样就会使事件处理延迟了 ，请参照loop函数 */
  // 3---->只有IO线程的事件回调中调用queueInLoop才不需要唤醒 ，因为handleEvent执行后，
  //下面接着执行doPendingFunctors
  if (!isInLoopThread() || callingPendingFunctors_)
  {
    wakeup();
  }
}
 
TimerId EventLoop::runAt(const Timestamp& time, const TimerCallback& cb)
{
  return timerQueue_->addTimer(cb, time, 0.0);
}
 
TimerId EventLoop::runAfter(double delay, const TimerCallback& cb)
{
  Timestamp time(addTime(Timestamp::now(), delay));
  return runAt(time, cb);
}
 
TimerId EventLoop::runEvery(double interval, const TimerCallback& cb)
{
  Timestamp time(addTime(Timestamp::now(), interval));
  return timerQueue_->addTimer(cb, time, interval);
}
 
void EventLoop::cancel(TimerId timerId)
{
  return timerQueue_->cancel(timerId);
}
 
void EventLoop::updateChannel(Channel* channel)
{
  assert(channel->ownerLoop() == this);
  assertInLoopThread();
  poller_->updateChannel(channel);
}
 
void EventLoop::removeChannel(Channel* channel)
{
  assert(channel->ownerLoop() == this);
  assertInLoopThread();
  if (eventHandling_)
  {
    assert(currentActiveChannel_ == channel ||
        std::find(activeChannels_.begin(), activeChannels_.end(), channel) == activeChannels_.end());
  }
  poller_->removeChannel(channel);
}
 
void EventLoop::abortNotInLoopThread()
{
  LOG_FATAL << "EventLoop::abortNotInLoopThread - EventLoop " << this
            << " was created in threadId_ = " << threadId_
            << ", current thread id = " <<  CurrentThread::tid();
}
 
/*
 
 
唤醒等待的线程
*/
void EventLoop::wakeup()
{
  uint64_t one = 1;
  //ssize_t n = sockets::write(wakeupFd_, &one, sizeof one);
  /*向wakeupFd_中写入数据*/
  ssize_t n = ::write(wakeupFd_, &one, sizeof one);
  if (n != sizeof one)
  {
    LOG_ERROR << "EventLoop::wakeup() writes " << n << " bytes instead of 8";
  }
}
 
// IO事件回调函数
void EventLoop::handleRead()
{
  uint64_t one = 1;
  //ssize_t n = sockets::read(wakeupFd_, &one, sizeof one);--->>muduolib
  ssize_t n = ::read(wakeupFd_, &one, sizeof one); // 这是先这样设定
  if (n != sizeof one)
  {
    LOG_ERROR << "EventLoop::handleRead() reads " << n << " bytes instead of 8";
  }
}
 
/*IO线程的回调函数集 处理函数*/
void EventLoop::doPendingFunctors()
{
  std::vector<Functor> functors;
  /*正在处于IO线程处理函数中*/
  callingPendingFunctors_ = true;
 
  {
    // 加锁
  MutexLockGuard lock(mutex_);
  //交换，顺带将pendingFunctors清空
  functors.swap(pendingFunctors_);
  }
 
  for (size_t i = 0; i < functors.size(); ++i)
  {
    functors[i]();
  }
  /*清空IO线程回调处理函数中 的状态*/
  callingPendingFunctors_ = false;
}
 
void EventLoop::printActiveChannels() const
{
  for (ChannelList::const_iterator it = activeChannels_.begin();
      it != activeChannels_.end(); ++it)
  {
    const Channel* ch = *it;
    LOG_TRACE << "{" << ch->reventsToString() << "} ";
  }
}
```

### 测试程序源代码

``` c++
#include <muduo/net/EventLoop.h>
//#include <muduo/net/EventLoopThread.h>
//#include <muduo/base/Thread.h>
 
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
EventLoop* g_loop;
int g_flag = 0;
 
void run4()
{
  printf("run4(): pid = %d, flag = %d\n", getpid(), g_flag);
  g_loop->quit();
}
 
void run3()
{
  printf("run3(): pid = %d, flag = %d\n", getpid(), g_flag);
 
  g_loop->runAfter(3, run4);
  g_flag = 3;
}
 
void run2()
{
  printf("run2(): pid = %d, flag = %d\n", getpid(), g_flag);
    /*在IO线程中直接调用queueInLoop，并且if (!isInLoopThread() || callingPendingFunctors_)时，
  不用唤醒（wakeup()）
    */
  g_loop->queueInLoop(run3);
}
 
void run1()
{
  g_flag = 1;
  printf("run1(): pid = %d, flag = %d\n", getpid(), g_flag);
  /*在IO线程中，调用runInLoop时，不用经过queueInLoop（）就可以直接调用Functors*/
  g_loop->runInLoop(run2);
  g_flag = 2;
}
 
int main()
{
  printf("main(): pid = %d, flag = %d\n", getpid(), g_flag);
 
  EventLoop loop;
  g_loop = &loop;
 
  loop.runAfter(2, run1);
  loop.loop();
  printf("main(): pid = %d, flag = %d\n", getpid(), g_flag);
}
```

### 测试输出

``` sh
ubuntu@ubuntu-virtual-machine:~/29/jmuduo$ ../build/debug/bin/reactor_test05
main(): pid = 28193, flag = 0
20131022 01:36:15.466526Z 28193 TRACE updateChannel fd = 4 events = 3 - EPollPoller.cc:104
20131022 01:36:15.467517Z 28193 TRACE EventLoop EventLoop created 0xBFEDB874 in thread 28193 - EventLoop.cc:62
20131022 01:36:15.467586Z 28193 TRACE updateChannel fd = 5 events = 3 - EPollPoller.cc:104
20131022 01:36:15.467898Z 28193 TRACE loop EventLoop 0xBFEDB874 start looping - EventLoop.cc:94
20131022 01:36:17.469365Z 28193 TRACE poll 1 events happended - EPollPoller.cc:65
20131022 01:36:17.482824Z 28193 TRACE printActiveChannels {4: IN }  - EventLoop.cc:257
20131022 01:36:17.483170Z 28193 TRACE readTimerfd TimerQueue::handleRead() 1 at 1382405777.482909 - TimerQueue.cc:62
run1(): pid = 28193, flag = 1
run2(): pid = 28193, flag = 1
run3(): pid = 28193, flag = 2
20131022 01:36:20.483953Z 28193 TRACE poll 1 events happended - EPollPoller.cc:65
20131022 01:36:20.484074Z 28193 TRACE printActiveChannels {4: IN }  - EventLoop.cc:257
20131022 01:36:20.484119Z 28193 TRACE readTimerfd TimerQueue::handleRead() 1 at 1382405780.484109 - TimerQueue.cc:62
run4(): pid = 28193, flag = 3
20131022 01:36:20.484229Z 28193 TRACE loop EventLoop 0xBFEDB874 stop looping - EventLoop.cc:119
main(): pid = 28193, flag = 3
ubuntu@ubuntu-virtual-machine:~/29/jmuduo$
```

## [30] EventLoopThread

任何一个线程，只要创建并运行了EventLoop，都称之为IO线程

IO线程不一定是主线程

muduo并发模型one loop per thread + threadpool

为了方便今后使用，定义了EventLoopThread类，该类封装了IO线程EventLoopThread创建了一个线程

在线程函数中创建了一个EvenLoop对象并调用EventLoop::loop





### EventLoopThread头文件

eventloopthread.h

``` c++

// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_EVENTLOOPTHREAD_H
#define MUDUO_NET_EVENTLOOPTHREAD_H
 
#include <muduo/base/Condition.h>
#include <muduo/base/Mutex.h>
#include <muduo/base/Thread.h>
 
#include <boost/noncopyable.hpp>
 
namespace muduo
{
namespace net
{
 
class EventLoop;
 
class EventLoopThread : boost::noncopyable
{
 public:
  typedef boost::function<void(EventLoop*)> ThreadInitCallback;
 
  EventLoopThread(const ThreadInitCallback& cb =ThreadInitCallback());
  ~EventLoopThread();
  EventLoop* startLoop();   // 启动线程，该线程就成为了IO线程
 
 private:
  void threadFunc();        // 线程函数 ，这个线程函数启动起来之后，会创建一个eventloop对象，loop_指针会指向这个对象
 
  EventLoop* loop_;         // loop_指针指向一个EventLoop对象，一个IO线程有且只有一个eventloop对象
  bool exiting_;
  Thread thread_;       //线程对象
  MutexLock mutex_;
  Condition cond_;
  ThreadInitCallback callback_;     // 回调函数在EventLoop::loop IO线程的事件循环之前被调用
};
 
}
}
 
#endif  // MUDUO_NET_EVENTLOOPTHREAD_H
```

### EventLoopThread源文件

eventloopthread.cc

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/net/EventLoopThread.h>
 
#include <muduo/net/EventLoop.h>
 
#include <boost/bind.hpp>
 
using namespace muduo;
using namespace muduo::net;
// typedef boost::function<void(EventLoop*)> ThreadInitCallback;
 
EventLoopThread::EventLoopThread(const ThreadInitCallback& cb)
  : loop_(NULL),
    exiting_(false),
    thread_(boost::bind(&EventLoopThread::threadFunc, this)),
    mutex_(),
    cond_(mutex_),
    callback_(cb)  //cb 默认为空
{
}
 
EventLoopThread::~EventLoopThread()
{
  exiting_ = true;
  loop_->quit();     // 退出IO线程，让IO线程的loop循环退出，从而退出了IO线程
  thread_.join();
}
 
/*启动线程，也就是启动loop函数EventLoopThread::threadFunc这个函数*/
EventLoop* EventLoopThread::startLoop()
{
  assert(!thread_.started());
  /*线程启动时会调用*/
  thread_.start();
 
  {
    MutexLockGuard lock(mutex_);
    /*eventloop对象创建完成*/
    while (loop_ == NULL)
    {
      cond_.wait();
    }
  }
 
  return loop_;
}
 
/*loop 的初始化函数*/
void EventLoopThread::threadFunc()
{
  EventLoop loop;
  /*创建eventloop对象*/
  if (callback_)
  {
    callback_(&loop);
  }
 
  {
    MutexLockGuard lock(mutex_);
    // loop_指针指向了一个栈上的对象，threadFunc函数退出之后，这个指针就失效了
    // threadFunc函数退出，就意味着线程退出了，EventLoopThread对象也就没有存在的价值了。
    // 因而不会有什么大的问题
    loop_ = &loop;
    cond_.notify();
  }
  /*启动loop（）函数*/
  loop.loop();
  //assert(exiting_);
}
```

###  测试程序

``` c++
#include <muduo/net/EventLoop.h>
#include <muduo/net/EventLoopThread.h>
 
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
void runInThread()
{
  printf("runInThread(): pid = %d, tid = %d\n",
         getpid(), CurrentThread::tid());
}
 
int main()
{
  printf("main(): pid = %d, tid = %d\n",
         getpid(), CurrentThread::tid());
 
  EventLoopThread loopThread;
  EventLoop* loop = loopThread.startLoop();
  // 异步调用runInThread，即将runInThread添加到loop对象所在IO线程，让该IO线程执行
  loop->runInLoop(runInThread);
  sleep(1);
  // runAfter内部也调用了runInLoop，所以这里也是异步调用
  loop->runAfter(2, runInThread);
  sleep(3);
  loop->quit();
 
  printf("exit main().\n");
}
 
/*
TimerId TimerQueue::addTimer(const TimerCallback& cb,
                             Timestamp when,
                             double interval)
{
  Timer* timer = new Timer(cb, when, interval);
 
  loop_->runInLoop(
      boost::bind(&TimerQueue::addTimerInLoop, this, timer));
 
  //addTimerInLoop(timer); 不能用这个，如果直接调用，那么addTimerInLoop断言失败
  return TimerId(timer, timer->sequence());
}
 
void TimerQueue::addTimerInLoop(Timer* timer)
{
  loop_->assertInLoopThread();
  // 插入一个定时器，有可能会使得最早到期的定时器发生改变
  bool earliestChanged = insert(timer);
 
  if (earliestChanged)
  {
    // 重置定时器的超时时刻(timerfd_settime)
    resetTimerfd(timerfd_, timer->expiration());
  }
}
 
bool TimerQueue::insert(Timer* timer)
{
  loop_->assertInLoopThread();
  assert(timers_.size() == activeTimers_.size());
  // 最早到期时间是否改变
  bool earliestChanged = false;
  Timestamp when = timer->expiration();
  TimerList::iterator it = timers_.begin();
  // 如果timers_为空或者when小于timers_中的最早到期时间
  if (it == timers_.end() || when < it->first)
  {
    earliestChanged = true;
  }
  {
    // 插入到timers_中
    std::pair<TimerList::iterator, bool> result
      = timers_.insert(Entry(when, timer));
    assert(result.second); (void)result;
  }
  {
    // 插入到activeTimers_中
    std::pair<ActiveTimerSet::iterator, bool> result
      = activeTimers_.insert(ActiveTimer(timer, timer->sequence()));
    assert(result.second); (void)result;
  }
 
  assert(timers_.size() == activeTimers_.size());
  return earliestChanged;
}
 
 
*/
```

### 程序输出

``` sh

ubuntu@ubuntu-virtual-machine:~/pro/30$ ./build/debug/bin/reactor_test06
main(): pid = 29731, tid = 29731
20131022 03:03:50.315042Z 29732 TRACE updateChannel fd = 4 events = 3 - EPollPoller.cc:104
20131022 03:03:50.315588Z 29732 TRACE EventLoop EventLoop created 0xB786BF94 in thread 29732 - EventLoop.cc:62
20131022 03:03:50.315643Z 29732 TRACE updateChannel fd = 5 events = 3 - EPollPoller.cc:104
20131022 03:03:50.315708Z 29732 TRACE loop EventLoop 0xB786BF94 start looping - EventLoop.cc:94
20131022 03:03:50.315976Z 29732 TRACE poll 1 events happended - EPollPoller.cc:65
20131022 03:03:50.316472Z 29732 TRACE printActiveChannels {5: IN }  - EventLoop.cc:257
runInThread(): pid = 29731, tid = 29732
20131022 03:03:51.316968Z 29732 TRACE poll 1 events happended - EPollPoller.cc:65
20131022 03:03:51.317092Z 29732 TRACE printActiveChannels {5: IN }  - EventLoop.cc:257
20131022 03:03:53.317414Z 29732 TRACE poll 1 events happended - EPollPoller.cc:65
20131022 03:03:53.317539Z 29732 TRACE printActiveChannels {4: IN }  - EventLoop.cc:257
20131022 03:03:53.317639Z 29732 TRACE readTimerfd TimerQueue::handleRead() 1 at 1382411033.317576 - TimerQueue.cc:62
runInThread(): pid = 29731, tid = 29732
exit main().
20131022 03:03:54.317901Z 29732 TRACE poll 1 events happended - EPollPoller.cc:65
20131022 03:03:54.317966Z 29732 TRACE printActiveChannels {5: IN }  - EventLoop.cc:257
20131022 03:03:54.317990Z 29732 TRACE loop EventLoop 0xB786BF94 stop looping - EventLoop.cc:119
ubuntu@ubuntu-virtual-machine:~/pro/30$
```

## [31] Socket封装


- Endian.h
  封装了字节序转换函数（全局函数，位于muduo::net::sockets名称空间中）。
- SocketsOps.h/ SocketsOps.cc
  封装了socket相关系统调用（全局函数，位于muduo::net::sockets名称空间中）。
- Socket.h/Socket.cc（Socket类）
  用RAII方法封装socket file descriptor
- InetAddress.h/InetAddress.cc（InetAddress类）
  网际地址sockaddr_in封装

### Endian头文件

endian.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_ENDIAN_H
#define MUDUO_NET_ENDIAN_H
 
#include <stdint.h>
#include <endian.h>
 
namespace muduo
{
namespace net
{
namespace sockets
{
 
// the inline assembler code makes type blur,
// so we disable warnings for a while.
#if __GNUC_MINOR__ >= 6
#pragma GCC diagnostic push
#endif
#pragma GCC diagnostic ignored "-Wconversion"
#pragma GCC diagnostic ignored "-Wold-style-cast"
inline uint64_t hostToNetwork64(uint64_t host64)
{
  return htobe64(host64);
}
 
inline uint32_t hostToNetwork32(uint32_t host32)
{
  return htobe32(host32);
}
 
inline uint16_t hostToNetwork16(uint16_t host16)
{
  return htobe16(host16);
}
 
inline uint64_t networkToHost64(uint64_t net64)
{
  return be64toh(net64);
}
 
inline uint32_t networkToHost32(uint32_t net32)
{
  return be32toh(net32);
}
 
inline uint16_t networkToHost16(uint16_t net16)
{
  return be16toh(net16);
}
#if __GNUC_MINOR__ >= 6
#pragma GCC diagnostic pop
#else
#pragma GCC diagnostic error "-Wconversion"
#pragma GCC diagnostic error "-Wold-style-cast"
#endif
 
 
}
}
}
 
#endif  // MUDUO_NET_ENDIAN_H

```

### InetAddress头文件

inetaddress.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_INETADDRESS_H
#define MUDUO_NET_INETADDRESS_H
 
#include <muduo/base/copyable.h>
#include <muduo/base/StringPiece.h>
 
#include <netinet/in.h>
 
namespace muduo
{
namespace net
{
 
///
/// Wrapper of sockaddr_in.
///
/// This is an POD interface class.
class InetAddress : public muduo::copyable
{
 public:
  /// Constructs an endpoint with given port number.
  /// Mostly used in TcpServer listening.
  // 仅仅指定port，不指定ip，则ip为INADDR_ANY（即0.0.0.0）
  explicit InetAddress(uint16_t port);
 
  /// Constructs an endpoint with given ip and port.
  /// @c ip should be "1.2.3.4"
  InetAddress(const StringPiece& ip, uint16_t port);
 
  /// Constructs an endpoint with given struct @c sockaddr_in
  /// Mostly used when accepting new connections
  InetAddress(const struct sockaddr_in& addr)
    : addr_(addr)
  { }
 
  string toIp() const;
  string toIpPort() const;
 
  // __attribute__ ((deprecated)) 表示该函数是过时的，被淘汰的
  // 这样使用该函数，在编译的时候，会发出警告
  /* 保留这个函数的原因是 为了 软件升级使用，最后不要调用这个函数，直接调用toIpPort就行了*/
  string toHostPort() const __attribute__ ((deprecated))
  { return toIpPort(); }
 
  // default copy/assignment are Okay
 
  const struct sockaddr_in& getSockAddrInet() const { return addr_; }
  void setSockAddrInet(const struct sockaddr_in& addr) { addr_ = addr; }
 
  uint32_t ipNetEndian() const { return addr_.sin_addr.s_addr; }
  uint16_t portNetEndian() const { return addr_.sin_port; }
 
 private:
  struct sockaddr_in addr_;
};
 
}
}
 
#endif  // MUDUO_NET_INETADDRESS_H

```

### InetAddress源文件

inetaddress.cc

``` c
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/net/InetAddress.h>
 
#include <muduo/net/Endian.h>
#include <muduo/net/SocketsOps.h>
 
#include <strings.h>  // bzero
#include <netinet/in.h>
 
#include <boost/static_assert.hpp>
 
// INADDR_ANY use (type)value casting.
#pragma GCC diagnostic ignored "-Wold-style-cast"
static const in_addr_t kInaddrAny = INADDR_ANY;
#pragma GCC diagnostic error "-Wold-style-cast"
 
//     /* Structure describing an Internet socket address.  */
//     struct sockaddr_in {
//         sa_family_t    sin_family; /* address family: AF_INET */
//         uint16_t       sin_port;   /* port in network byte order */
//         struct in_addr sin_addr;   /* internet address */
//     };
 
//     /* Internet address. */
//     typedef uint32_t in_addr_t;
//     struct in_addr {
//         in_addr_t       s_addr;     /* address in network byte order */
//     };
 
using namespace muduo;
using namespace muduo::net;
 
BOOST_STATIC_ASSERT(sizeof(InetAddress) == sizeof(struct sockaddr_in));
 
InetAddress::InetAddress(uint16_t port)
{
  bzero(&addr_, sizeof addr_);
  addr_.sin_family = AF_INET;
  addr_.sin_addr.s_addr = sockets::hostToNetwork32(kInaddrAny);
  addr_.sin_port = sockets::hostToNetwork16(port);
}
 
InetAddress::InetAddress(const StringPiece& ip, uint16_t port)
{
  bzero(&addr_, sizeof addr_);
  sockets::fromIpPort(ip.data(), port, &addr_);
}
 
string InetAddress::toIpPort() const
{
  char buf[32];
  sockets::toIpPort(buf, sizeof buf, addr_);
  return buf;
}
 
string InetAddress::toIp() const
{
  char buf[32];
  sockets::toIp(buf, sizeof buf, addr_);
  return buf;
}
```

### Sokcet头文件

Socket.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is an internal header file, you should not include this.
 
#ifndef MUDUO_NET_SOCKET_H
#define MUDUO_NET_SOCKET_H
 
#include <boost/noncopyable.hpp>
 
namespace muduo
{
///
/// TCP networking.
///
namespace net
{
 
class InetAddress;
 
///
/// Wrapper of socket file descriptor.
///
/// It closes the sockfd when desctructs.
/// It's thread safe, all operations are delagated to OS.
class Socket : boost::noncopyable
{
 public:
  explicit Socket(int sockfd)
    : sockfd_(sockfd) //已已经初始化的sockfd
  { }
 
  // Socket(Socket&&) // move constructor in C++11
  ~Socket();
 
  int fd() const { return sockfd_; }
 
  /// abort if address in use
  void bindAddress(const InetAddress& localaddr);
  /// abort if address in use
  void listen();
 
  /// On success, returns a non-negative integer that is
  /// a descriptor for the accepted socket, which has been
  /// set to non-blocking and close-on-exec. *peeraddr is assigned.
  /// On error, -1 is returned, and *peeraddr is untouched.
  int accept(InetAddress* peeraddr);
 
  void shutdownWrite();
 
  ///
  /// Enable/disable TCP_NODELAY (disable/enable Nagle's algorithm).
  ///
  // Nagle算法可以一定程度上避免网络拥塞
  // TCP_NODELAY选项可以禁用Nagle算法
  // 禁用Nagle算法，可以避免连续发包出现延迟，这对于编写低延迟的网络服务很重要
  void setTcpNoDelay(bool on);
 
  ///
  /// Enable/disable SO_REUSEADDR
  ///
  void setReuseAddr(bool on);
 
  ///
  /// Enable/disable SO_KEEPALIVE
  ///
  // TCP keepalive是指定期探测连接是否存在，如果应用层有心跳的话，这个选项不是必需要设置的
  void setKeepAlive(bool on);
 
 private:
  const int sockfd_;
};
 
}
}
#endif  // MUDUO_NET_SOCKET_H
```

### Socket源文件

Socket.cc

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/net/Socket.h>
 
#include <muduo/net/InetAddress.h>
#include <muduo/net/SocketsOps.h>
 
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <strings.h>  // bzero
 
using namespace muduo;
using namespace muduo::net;
 
Socket::~Socket()
{
  sockets::close(sockfd_);
}
 
void Socket::bindAddress(const InetAddress& addr)
{
  sockets::bindOrDie(sockfd_, addr.getSockAddrInet());
}
 
void Socket::listen()
{
  sockets::listenOrDie(sockfd_);
}
 
int Socket::accept(InetAddress* peeraddr)
{
  struct sockaddr_in addr;
  bzero(&addr, sizeof addr);
  int connfd = sockets::accept(sockfd_, &addr);
  if (connfd >= 0)
  {
    peeraddr->setSockAddrInet(addr);
  }
  return connfd;
}
 
void Socket::shutdownWrite()
{
  sockets::shutdownWrite(sockfd_);
}
 
void Socket::setTcpNoDelay(bool on)
{
  int optval = on ? 1 : 0;
  ::setsockopt(sockfd_, IPPROTO_TCP, TCP_NODELAY,
               &optval, sizeof optval);
  // FIXME CHECK
}
 
void Socket::setReuseAddr(bool on)
{
  int optval = on ? 1 : 0;
  ::setsockopt(sockfd_, SOL_SOCKET, SO_REUSEADDR,
               &optval, sizeof optval);
  // FIXME CHECK
}
 
void Socket::setKeepAlive(bool on)
{
  int optval = on ? 1 : 0;
  ::setsockopt(sockfd_, SOL_SOCKET, SO_KEEPALIVE,
               &optval, sizeof optval);
  // FIXME CHECK
}

```

### 程序程序

``` c++
#include <muduo/net/InetAddress.h>
 
//#define BOOST_TEST_MODULE InetAddressTest
#define BOOST_TEST_MAIN
#define BOOST_TEST_DYN_LINK
#include <boost/test/unit_test.hpp>
 
using muduo::string;
using muduo::net::InetAddress;
 
BOOST_AUTO_TEST_CASE(testInetAddress)
{
  InetAddress addr1(1234);
  BOOST_CHECK_EQUAL(addr1.toIp(), string("0.0.0.0"));
  BOOST_CHECK_EQUAL(addr1.toIpPort(), string("0.0.0.0:1234"));
 
  InetAddress addr2("1.2.3.4", 8888);
  BOOST_CHECK_EQUAL(addr2.toIp(), string("1.2.3.4"));
  BOOST_CHECK_EQUAL(addr2.toIpPort(), string("1.2.3.4:8888"));
 
  InetAddress addr3("255.255.255.255", 65535);
  BOOST_CHECK_EQUAL(addr3.toIp(), string("255.255.255.255"));
  BOOST_CHECK_EQUAL(addr3.toIpPort(), string("255.255.255.255:65535"));
}
```

## [32] Acceptor

- Acceptor用于accept(2)接受TCP连接
- Acceptor的数据成员包括`Socket`、`Channel`，`Acceptor`的`socket`是`listening socket`（即`server socket`）。`Channel`用于观察此`socket`的`readable`事件，并回调`Accptor::handleRead()`，后者调用`accept(2)`来接受新连接，并回调用户`callback`。

### Acceptor的头文件

Acceptor.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is an internal header file, you should not include this.
 
#ifndef MUDUO_NET_ACCEPTOR_H
#define MUDUO_NET_ACCEPTOR_H
 
#include <boost/function.hpp>
#include <boost/noncopyable.hpp>
 
#include <muduo/net/Channel.h>
#include <muduo/net/Socket.h>
 
namespace muduo
{
namespace net
{
 
class EventLoop;
class InetAddress;
 
///
/// Acceptor of incoming TCP connections.
///
class Acceptor : boost::noncopyable
{
 public:
  typedef boost::function<void (int sockfd,
                                const InetAddress&)> NewConnectionCallback;
 
  Acceptor(EventLoop* loop, const InetAddress& listenAddr);
  ~Acceptor();
 
  void setNewConnectionCallback(const NewConnectionCallback& cb)
  { newConnectionCallback_ = cb; }
 
  bool listenning() const { return listenning_; }
  void listen();
 
 private:
  /*这是监听套接字的可读事件的套接字*/
  void handleRead();
 
  EventLoop* loop_;
  Socket acceptSocket_; /*监听套接字*/
  ;/*通道 会观察acceptSocket_监听套接字的可读事件，这个channel所属的eventloop是loop_*/
  Channel acceptChannel_;
  NewConnectionCallback newConnectionCallback_;
  /*是否处于监听的状态*/
  bool listenning_;
  int idleFd_;
};
 
}
}
 
#endif  // MUDUO_NET_ACCEPTOR_H
```

### Acceptor源文件

Acceptor.cc

```  c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
// Author: Shuo Chen (chenshuo at chenshuo dot com)
#include <muduo/net/Acceptor.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
#include <muduo/net/SocketsOps.h>
 
#include <boost/bind.hpp>
 
#include <errno.h>
#include <fcntl.h>
//#include <sys/types.h>
//#include <sys/stat.h>
 
using namespace muduo;
using namespace muduo::net;
 
Acceptor::Acceptor(EventLoop* loop, const InetAddress& listenAddr)
  : loop_(loop),
  /*创建一个监听套接字*/
    acceptSocket_(sockets::createNonblockingOrDie()),
  /*acceptchannel 关注acceptSocket_套接字的事件 */
    acceptChannel_(loop, acceptSocket_.fd()),
    listenning_(false),
    /*预先准备一个文件描述符*/
    idleFd_(::open("/dev/null", O_RDONLY | O_CLOEXEC))
{
  assert(idleFd_ >= 0);
  /*设置地址重复利用*/
  acceptSocket_.setReuseAddr(true);
  /*绑定地址*/
  acceptSocket_.bindAddress(listenAddr);
  /*设置监听套接字“读”的套接字*/
  acceptChannel_.setReadCallback(
      boost::bind(&Acceptor::handleRead, this));
}
 
/*虚构函数*/
Acceptor::~Acceptor()
{
  /*先取消 acceptSocket_的所有事件，然后才移除*/
  acceptChannel_.disableAll();
  acceptChannel_.remove();
  ::close(idleFd_);
}
/*监听 监听套接字*/
void Acceptor::listen()
{/*断言在IO线程中*/
  loop_->assertInLoopThread();
 
  listenning_ = true;
  acceptSocket_.listen();
  /*关注acceptSocket_的可读事件*/
  acceptChannel_.enableReading();
}
 
/*监听套接字的可读事件的回调函数*/
void Acceptor::handleRead()
{
  /*断言在IO线程中*/
  loop_->assertInLoopThread();
  /*创建一个对等方的地址，这是客户端的地址*/
  InetAddress peerAddr(0);
  //FIXME loop until no more
  /*对等连接套接字*/
  int connfd = acceptSocket_.accept(&peerAddr);
  if (connfd >= 0)
  {
    // string hostport = peerAddr.toIpPort();
    // LOG_TRACE << "Accepts of " << hostport;
    /*回调上层（应用层）的回调函数*/
    if (newConnectionCallback_)
    {
      newConnectionCallback_(connfd, peerAddr);
    }
    /*如果应用层没有回调函数，则直接关闭此连接套接字*/
    else
    {
      sockets::close(connfd);
    }
  }
  else
  {
    // Read the section named "The special problem of
    // accept()ing when you can't" in libev's doc.
    // By Marc Lehmann, author of livev.
    /*如果是文件符超出限制*/
    if (errno == EMFILE)
    {
      /**/
      ::close(idleFd_);
      /*因为这是电平触发，所以为了防止一直触发，可以先接受，然后关闭掉*/
      idleFd_ = ::accept(acceptSocket_.fd(), NULL, NULL);
      ::close(idleFd_);
      idleFd_ = ::open("/dev/null", O_RDONLY | O_CLOEXEC);
    }
  }
}
```

### 测试程序

``` c++
#include <muduo/net/Acceptor.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
#include <muduo/net/SocketsOps.h>
 
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
void newConnection(int sockfd, const InetAddress& peerAddr)
{
  printf("newConnection(): accepted a new connection from %s\n",
         peerAddr.toIpPort().c_str());
  ::write(sockfd, "How are you?\n", 13);
  sockets::close(sockfd);
}
 
int main()
{
  printf("main(): pid = %d\n", getpid());
 
  InetAddress listenAddr(8888);
  EventLoop loop;
 
  Acceptor acceptor(&loop, listenAddr);
  acceptor.setNewConnectionCallback(newConnection);
  acceptor.listen();
 
  loop.loop();
}
```

### 程序输出

``` c
ubuntu@ubuntu-virtual-machine:~$ telnet 127.0.0.1 8888
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
How Are You ?
Connection closed by foreign host.
ubuntu@ubuntu-virtual-machine:~$ telnet 127.0.0.1 8888
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
How Are You ?
Connection closed by foreign host.
ubuntu@ubuntu-virtual-machine:~$
```
