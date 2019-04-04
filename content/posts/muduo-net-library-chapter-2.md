---
title: muduo net library chapter 2
date: 2015-07-01 00:15:03
tags:
- muduo
categories:
- muduo 
- network program

description: muduo 是一个linux 网络库，使用poll和epoll模式。 这些内容是本人在大学的时候，通过  << c++教程网 >>  所学的知识笔记. 已经过去好几年了，版本也是比较老的了，属于入门级别的教程； 如有错误，纯属正常!!!

---

## [30] EventLoopThread

- 任何一个线程，只要创建并运行了EventLoop，都称之为IO线程
- IO线程不一定是主线程
- muduo并发模型one loop per thread + threadpool
- 为了方便今后使用，定义了EventLoopThread类，该类封装了IO线程EventLoopThread创建了一个线程
- 在线程函数中创建了一个EvenLoop对象并调用EventLoop::loop

### EventLoopThread头文件

eventloopthread.h

```c++
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

```c++
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

### 测试程序

```bash
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

```c++
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

## [33] TcpConnection生存期管理

时序图

<center>![http://laohanlinux.github.io/images/img/blog/muduo-base-library-2/34.png]</center>

主要的改变时TcpConnection 和 channel 连个对象

### TcpConnection

```c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_TCPCONNECTION_H
#define MUDUO_NET_TCPCONNECTION_H
 
#include <muduo/base/Mutex.h>
#include <muduo/base/StringPiece.h>
#include <muduo/base/Types.h>
#include <muduo/net/Callbacks.h>
//#include <muduo/net/Buffer.h>
#include <muduo/net/InetAddress.h>
 
//#include <boost/any.hpp>
#include <boost/enable_shared_from_this.hpp>
#include <boost/noncopyable.hpp>
#include <boost/scoped_ptr.hpp>
#include <boost/shared_ptr.hpp>
 
namespace muduo
{
namespace net
{
 
class Channel;
class EventLoop;
class Socket;
 
 
///
/// TCP connection, for both client and server usage.
///
/// This is an interface class, so don't expose too much details.
class TcpConnection : boost::noncopyable,
                      public boost::enable_shared_from_this<TcpConnection>
{
 public:
  /// Constructs a TcpConnection with a connected sockfd
  ///
  /// User should not create this object.
  TcpConnection(EventLoop* loop,
                const string& name,
                int sockfd,
                const InetAddress& localAddr,
                const InetAddress& peerAddr);
  ~TcpConnection();
 
  EventLoop* getLoop() const { return loop_; }
  const string& name() const { return name_; }
  const InetAddress& localAddress() { return localAddr_; }
  const InetAddress& peerAddress() { return peerAddr_; }
  bool connected() const { return state_ == kConnected; }
 
  void setConnectionCallback(const ConnectionCallback& cb)
  { connectionCallback_ = cb; }
 
  void setMessageCallback(const MessageCallback& cb)
  { messageCallback_ = cb; }
 
  /// Internal use only.
  void setCloseCallback(const CloseCallback& cb)
  { closeCallback_ = cb; }
 
  // called when TcpServer accepts a new connection
  void connectEstablished();   // should be called only once
  // called when TcpServer has removed me from its map
  void connectDestroyed();  // should be called only once
 
 private:
  enum StateE { kDisconnected, kConnecting, kConnected, kDisconnecting };
  void handleRead(Timestamp receiveTime);
  void handleClose();
  void handleError();
  void setState(StateE s) { state_ = s; }
 
  EventLoop* loop_;         // 所属EventLoop
  string name_;             // 连接名
  StateE state_;  // FIXME: use atomic variable
  // we don't expose those classes to client.
  boost::scoped_ptr<Socket> socket_;
  boost::scoped_ptr<Channel> channel_;
  InetAddress localAddr_;
  InetAddress peerAddr_;
  ConnectionCallback connectionCallback_;
  MessageCallback messageCallback_;
  /*内部断开的回调函数*/
  CloseCallback closeCallback_;
};
 
typedef boost::shared_ptr<TcpConnection> TcpConnectionPtr;
 
}
}
 
#endif  // MUDUO_NET_TCPCONNECTION_H
```

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/net/TcpConnection.h>
 
#include <muduo/base/Logging.h>
#include <muduo/net/Channel.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/Socket.h>
#include <muduo/net/SocketsOps.h>
 
#include <boost/bind.hpp>
 
#include <errno.h>
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
/*
void muduo::net::defaultConnectionCallback(const TcpConnectionPtr& conn)
{
  LOG_TRACE << conn->localAddress().toIpPort() << " -> "
            << conn->peerAddress().toIpPort() << " is "
            << (conn->connected() ? "UP" : "DOWN");
}
 
void muduo::net::defaultMessageCallback(const TcpConnectionPtr&,
                                        Buffer* buf,
                                        Timestamp)
{
  buf->retrieveAll();
}
*/
TcpConnection::TcpConnection(EventLoop* loop,
                             const string& nameArg,
                             int sockfd,
                             const InetAddress& localAddr,
                             const InetAddress& peerAddr)
  : loop_(CHECK_NOTNULL(loop)),
    name_(nameArg),
    state_(kConnecting),
    socket_(new Socket(sockfd)),
    channel_(new Channel(loop, sockfd)),
    localAddr_(localAddr),
    peerAddr_(peerAddr)/*,
    highWaterMark_(64*1024*1024)*/
{
  // 通道可读事件到来的时候，回调TcpConnection::handleRead，_1是事件发生时间
  channel_->setReadCallback(
      boost::bind(&TcpConnection::handleRead, this, _1));
  // 连接关闭，回调TcpConnection::handleClose
  channel_->setCloseCallback(
      boost::bind(&TcpConnection::handleClose, this));
  // 发生错误，回调TcpConnection::handleError
  channel_->setErrorCallback(
      boost::bind(&TcpConnection::handleError, this));
  LOG_DEBUG << "TcpConnection::ctor[" <<  name_ << "] at " << this
            << " fd=" << sockfd;
  socket_->setKeepAlive(true);
}
 
TcpConnection::~TcpConnection()
{
  LOG_DEBUG << "TcpConnection::dtor[" <<  name_ << "] at " << this
            << " fd=" << channel_->fd();
}
 
void TcpConnection::connectEstablished()
{
  loop_->assertInLoopThread();
  assert(state_ == kConnecting);
  setState(kConnected);
  LOG_TRACE << "[3] usecount=" << shared_from_this().use_count();
  channel_->tie(shared_from_this()); /*弱引用，不会使指针引用计算加一*/
  channel_->enableReading(); // TcpConnection所对应的通道加入到Poller关注
 
  /*用户的回调函数*/
  connectionCallback_(shared_from_this());
  LOG_TRACE << "[4] usecount=" << shared_from_this().use_count();
}
 
void TcpConnection::connectDestroyed()
{
  loop_->assertInLoopThread();
  if (state_ == kConnected)
  {
    setState(kDisconnected);
    channel_->disableAll();
 
    connectionCallback_(shared_from_this());
  }
  channel_->remove();
}
 
void TcpConnection::handleRead(Timestamp receiveTime)
{
  /*
  loop_->assertInLoopThread();
  int savedErrno = 0;
  ssize_t n = inputBuffer_.readFd(channel_->fd(), &savedErrno);
  if (n > 0)
  {
    messageCallback_(shared_from_this(), &inputBuffer_, receiveTime);
  }
  else if (n == 0)
  {
    handleClose();
  }
  else
  {
    errno = savedErrno;
    LOG_SYSERR << "TcpConnection::handleRead";
    handleError();
  }
  */
  loop_->assertInLoopThread();
  int savedErrno = 0;
  char buf[65536];
  ssize_t n = ::read(channel_->fd(), buf, sizeof buf);
  if (n > 0)
  {
    messageCallback_(shared_from_this(), buf, n);
  }
  /*如果是连接断开（连接断开也是一种可读事件），调用handclose*/
  else if (n == 0)
  {
    handleClose();
  }
  else
  {
    errno = savedErrno;
    LOG_SYSERR << "TcpConnection::handleRead";
    handleError();
  }
 
}
 
/*连接断开的内部回调函数*/
void TcpConnection::handleClose()
{
  loop_->assertInLoopThread();
  LOG_TRACE << "fd = " << channel_->fd() << " state = " << state_;
  assert(state_ == kConnected || state_ == kDisconnecting);
  // we don't close fd, leave it to dtor, so we can find leaks easily.
  setState(kDisconnected);
  channel_->disableAll();
 
  TcpConnectionPtr guardThis(shared_from_this());
  connectionCallback_(guardThis);       // 这一行，可以不调用
  LOG_TRACE << "[7] usecount=" << guardThis.use_count();
  // must be the last line
  closeCallback_(guardThis);    // 调用TcpServer::removeConnection
  LOG_TRACE << "[11] usecount=" << guardThis.use_count();
}
 
void TcpConnection::handleError()
{
  int err = sockets::getSocketError(channel_->fd());
  LOG_ERROR << "TcpConnection::handleError [" << name_
            << "] - SO_ERROR = " << err << " " << strerror_tl(err);
}
```

### Channel

```c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is an internal header file, you should not include this.
 
#ifndef MUDUO_NET_CHANNEL_H
#define MUDUO_NET_CHANNEL_H
 
#include <boost/function.hpp>
#include <boost/noncopyable.hpp>
#include <boost/shared_ptr.hpp>
#include <boost/weak_ptr.hpp>
 
#include <muduo/base/Timestamp.h>
 
namespace muduo
{
namespace net
{
 
class EventLoop;
 
///
/// A selectable I/O channel.
///
/// This class doesn't own the file descriptor.
/// The file descriptor could be a socket,
/// an eventfd, a timerfd, or a signalfd
class Channel : boost::noncopyable
{
 public:
  typedef boost::function<void()> EventCallback;
  typedef boost::function<void(Timestamp)> ReadEventCallback;
 
  Channel(EventLoop* loop, int fd);
  ~Channel();
 
  void handleEvent(Timestamp receiveTime);
  void setReadCallback(const ReadEventCallback& cb)
  { readCallback_ = cb; }
  void setWriteCallback(const EventCallback& cb)
  { writeCallback_ = cb; }
  void setCloseCallback(const EventCallback& cb)
  { closeCallback_ = cb; }
  void setErrorCallback(const EventCallback& cb)
  { errorCallback_ = cb; }
 
  /// Tie this channel to the owner object managed by shared_ptr,
  /// prevent the owner object being destroyed in handleEvent.
  void tie(const boost::shared_ptr<void>&);
 
  int fd() const { return fd_; }
  int events() const { return events_; }
  void set_revents(int revt) { revents_ = revt; } // used by pollers
  // int revents() const { return revents_; }
  bool isNoneEvent() const { return events_ == kNoneEvent; }
 
  void enableReading() { events_ |= kReadEvent; update(); }
  // void disableReading() { events_ &= ~kReadEvent; update(); }
  void enableWriting() { events_ |= kWriteEvent; update(); }
  void disableWriting() { events_ &= ~kWriteEvent; update(); }
  void disableAll() { events_ = kNoneEvent; update(); }
  bool isWriting() const { return events_ & kWriteEvent; }
 
  // for Poller
  int index() { return index_; }
  void set_index(int idx) { index_ = idx; }
 
  // for debug
  string reventsToString() const;
 
  void doNotLogHup() { logHup_ = false; }
 
  EventLoop* ownerLoop() { return loop_; }
  void remove();
 
 private:
  void update();
  void handleEventWithGuard(Timestamp receiveTime);
 
  static const int kNoneEvent;
  static const int kReadEvent;
  static const int kWriteEvent;
 
  EventLoop* loop_;         // 所属EventLoop
  const int  fd_;           // 文件描述符，但不负责关闭该文件描述符
  int        events_;       // 关注的事件
  int        revents_;      // poll/epoll返回的事件
  int        index_;        // used by Poller.表示在poll的事件数组中的序号
  bool       logHup_;       // for POLLHUP
 
  /*弱引用指针*/
  boost::weak_ptr<void> tie_;
  bool tied_;
  bool eventHandling_;      // 是否处于处理事件中
  ReadEventCallback readCallback_;
  EventCallback writeCallback_;
  EventCallback closeCallback_;
  EventCallback errorCallback_;
};
 
}
}
#endif  // MUDUO_NET_CHANNEL_H
```
``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/base/Logging.h>
#include <muduo/net/Channel.h>
#include <muduo/net/EventLoop.h>
 
#include <sstream>
 
#include <poll.h>
 
using namespace muduo;
using namespace muduo::net;
 
const int Channel::kNoneEvent = 0;
const int Channel::kReadEvent = POLLIN | POLLPRI;
const int Channel::kWriteEvent = POLLOUT;
 
Channel::Channel(EventLoop* loop, int fd__)
  : loop_(loop),
    fd_(fd__),
    events_(0),
    revents_(0),
    index_(-1),
    logHup_(true),
    tied_(false),
    eventHandling_(false)
{
}
 
Channel::~Channel()
{
  assert(!eventHandling_);
}
 
void Channel::tie(const boost::shared_ptr<void>& obj)
{
  tie_ = obj;
  tied_ = true;
}
 
void Channel::update()
{
  loop_->updateChannel(this);
}
 
// 调用这个函数之前确保调用disableAll
void Channel::remove()
{
  assert(isNoneEvent());
  loop_->removeChannel(this);
}
 
void Channel::handleEvent(Timestamp receiveTime)
{
  boost::shared_ptr<void> guard;
  if (tied_)
  {
    guard = tie_.lock();
    if (guard)
    {
      LOG_TRACE << "[6] usecount=" << guard.use_count();
      handleEventWithGuard(receiveTime);
      LOG_TRACE << "[12] usecount=" << guard.use_count();
    }
  }
  else
  {
    handleEventWithGuard(receiveTime);
  }
}
 
void Channel::handleEventWithGuard(Timestamp receiveTime)
{
  eventHandling_ = true;
  if ((revents_ & POLLHUP) && !(revents_ & POLLIN))
  {
    if (logHup_)
    {
      LOG_WARN << "Channel::handle_event() POLLHUP";
    }
    if (closeCallback_) closeCallback_();
  }
 
  if (revents_ & POLLNVAL)
  {
    LOG_WARN << "Channel::handle_event() POLLNVAL";
  }
 
  if (revents_ & (POLLERR | POLLNVAL))
  {
    if (errorCallback_) errorCallback_();
  }
  if (revents_ & (POLLIN | POLLPRI | POLLRDHUP))
  {
    if (readCallback_) readCallback_(receiveTime);
  }
  if (revents_ & POLLOUT)
  {
    if (writeCallback_) writeCallback_();
  }
  eventHandling_ = false;
}
 
string Channel::reventsToString() const
{
  std::ostringstream oss;
  oss << fd_ << ": ";
  if (revents_ & POLLIN)
    oss << "IN ";
  if (revents_ & POLLPRI)
    oss << "PRI ";
  if (revents_ & POLLOUT)
    oss << "OUT ";
  if (revents_ & POLLHUP)
    oss << "HUP ";
  if (revents_ & POLLRDHUP)
    oss << "RDHUP ";
  if (revents_ & POLLERR)
    oss << "ERR ";
  if (revents_ & POLLNVAL)
    oss << "NVAL ";
 
  return oss.str().c_str();
}
```

### 测试程序

```c++
#include <muduo/net/TcpServer.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
 
#include <boost/bind.hpp>
 
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
class TestServer
{
 public:
  TestServer(EventLoop* loop,
             const InetAddress& listenAddr)
    : loop_(loop),
      server_(loop, listenAddr, "TestServer")
  {
    server_.setConnectionCallback(
        boost::bind(&TestServer::onConnection, this, _1));
    server_.setMessageCallback(
        boost::bind(&TestServer::onMessage, this, _1, _2, _3));
  }
 
  void start()
  {
      server_.start();
  }
 
 private:
  void onConnection(const TcpConnectionPtr& conn)
  {
    if (conn->connected())
    {
      printf("onConnection(): new connection [%s] from %s\n",
             conn->name().c_str(),
             conn->peerAddress().toIpPort().c_str());
    }
    //增加了对等方关闭时的回调处理函数
    else
    {
      printf("onConnection(): connection [%s] is down\n",
             conn->name().c_str());
    }
  }
 
  void onMessage(const TcpConnectionPtr& conn,
                   const char* data,
                   ssize_t len)
  {
    printf("onMessage(): received %zd bytes from connection [%s]\n",
           len, conn->name().c_str());
  }
 
  EventLoop* loop_;
  TcpServer server_;
};
 
 
int main()
{
  printf("main(): pid = %d\n", getpid());
 
  InetAddress listenAddr(8888);
  EventLoop loop;
 
  TestServer server(&loop, listenAddr);
  server.start();
 
  loop.loop();
}
```
