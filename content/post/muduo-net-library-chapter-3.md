---
title: muduo net library chapter 3
date: 2015-08-01 00:15:03
tags:
- muduo
categories:
- muduo 
- network program

description: muduo 是一个linux 网络库，使用poll和epoll模式。 这些内容是本人在大学的时候，通过  << c++教程网 >>  所学的知识笔记. 已经过去好几年了，版本也是比较老的了，属于入门级别的教程； 如有错误，纯属正常!!!

---

## [34] TcpServer/TcpConnection

``` c++
Acceptor类的主要功能是socket、bind、listen
般来说，在上层应用程序中，我们不直接使用Acceptor，而是把它作为TcpServer的成员
TcpServer还包含了一个TcpConnection列表
TcpConnection与Acceptor类似，有两个重要的数据成员，Socket与Channel 
```

### 时序图

<center> ![](https://i.loli.net/2019/05/03/5ccbe089cae9f.png) </center>

### TcpServer头文件


TcpServer.h

```c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_TCPSERVER_H
#define MUDUO_NET_TCPSERVER_H
 
#include <muduo/base/Types.h>
#include <muduo/net/TcpConnection.h>
 
#include <map>
#include <boost/noncopyable.hpp>
#include <boost/scoped_ptr.hpp>
 
namespace muduo
{
namespace net
{
 
class Acceptor;
class EventLoop;
 
///
/// TCP server, supports single-threaded and thread-pool models.
///
/// This is an interface class, so don't expose too much details.
class TcpServer : boost::noncopyable
{
 public:
  //typedef boost::function<void(EventLoop*)> ThreadInitCallback;
 
  //TcpServer(EventLoop* loop, const InetAddress& listenAddr);
  TcpServer(EventLoop* loop,
            const InetAddress& listenAddr,
            const string& nameArg);
  ~TcpServer();  // force out-line dtor, for scoped_ptr members.
 
  /*返回服务器的名称*/
  const string& hostport() const { return hostport_; }
  /*返回服务器的监听端口*/
  const string& name() const { return name_; }
 
  /// Starts the server if it's not listenning.
  ///
  /// It's harmless to call it multiple times.
  /// Thread safe.
  void start();
 
  /// Set connection callback.
  /// Not thread safe.
  // 设置连接到来或者连接关闭回调函数
  void setConnectionCallback(const ConnectionCallback& cb)
  { connectionCallback_ = cb; }
 
  /// Set message callback.
  /// Not thread safe.
  // 设置消息到来回调函数
  void setMessageCallback(const MessageCallback& cb)
  { messageCallback_ = cb; }
 
 
 private:
  /// Not thread safe, but in loop
  // Acceptor::handleRead函数中会回调用TcpServer::newConnection
  // _1对应的是socket文件描述符，_2对应的是对等方的地址(InetAddress)
  //应该还在IO线程里面
  // newConnection和connectionCallback_都是连接回调函数，但是connectionCallback_
  // 是给应用程序使用的，newConnection是库函数，不会暴露给用户。
  // 其实就是newConnection函数主要负责注册connectionCallback_ 函数
  void newConnection(int sockfd, const InetAddress& peerAddr);
 
  typedef std::map<string, TcpConnectionPtr> ConnectionMap;
 
  EventLoop* loop_;  // the acceptor's loop，不一定是连接所属的eventloop
  const string hostport_;       // 服务端口
  const string name_;           // 服务名
  boost::scoped_ptr<Acceptor> acceptor_; // avoid revealing Acceptor
  /*连接到来的回调函数*/
  ConnectionCallback connectionCallback_;
  /*消息到来的回调函数*/
  MessageCallback messageCallback_;
  bool started_; /*连接是否启动*/
  // always in loop thread
  int nextConnId_;              // 下一个连接ID
  ConnectionMap connections_;   // 连接列表
};
 
}
}
 
#endif  // MUDUO_NET_TCPSERVER_H
```

### TcpServer源文件

TcpServer.cc

```c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/net/TcpServer.h>
 
#include <muduo/base/Logging.h>
#include <muduo/net/Acceptor.h>
#include <muduo/net/EventLoop.h>
//#include <muduo/net/EventLoopThreadPool.h>
#include <muduo/net/SocketsOps.h>
 
#include <boost/bind.hpp>
 
#include <stdio.h>  // snprintf
 
using namespace muduo;
using namespace muduo::net;
 
TcpServer::TcpServer(EventLoop* loop,
                     const InetAddress& listenAddr,
                     const string& nameArg)
  : loop_(CHECK_NOTNULL(loop)),
    hostport_(listenAddr.toIpPort()),
    name_(nameArg),
    acceptor_(new Acceptor(loop, listenAddr)),
    /*threadPool_(new EventLoopThreadPool(loop)),
    connectionCallback_(defaultConnectionCallback),
    messageCallback_(defaultMessageCallback),*/
    started_(false),
    nextConnId_(1)
{
  // Acceptor::handleRead函数中会回调用TcpServer::newConnection
  // _1对应的是socket文件描述符，_2对应的是对等方的地址(InetAddress)
  acceptor_->setNewConnectionCallback(
      boost::bind(&TcpServer::newConnection, this, _1, _2));
}
 
TcpServer::~TcpServer()
{
  loop_->assertInLoopThread();
  LOG_TRACE << "TcpServer::~TcpServer [" << name_ << "] destructing";
}
 
// 该函数多次调用是无害的
// 该函数可以跨线程调用
void TcpServer::start()
{
  if (!started_)
  {
    started_ = true;
  }
 
  /*如果还没有被执行，那么让他在IO线程中执行listen函数*/
  if (!acceptor_->listenning())
  {
    // get_pointer返回原生指针
    loop_->runInLoop(
        boost::bind(&Acceptor::listen, get_pointer(acceptor_)));
  }
}
 
void TcpServer::newConnection(int sockfd, const InetAddress& peerAddr)
{
  loop_->assertInLoopThread();
  char buf[32];
  snprintf(buf, sizeof buf, ":%s#%d", hostport_.c_str(), nextConnId_);
  ++nextConnId_;
  string connName = name_ + buf;
 
  LOG_INFO << "TcpServer::newConnection [" << name_
           << "] - new connection [" << connName
           << "] from " << peerAddr.toIpPort();
  InetAddress localAddr(sockets::getLocalAddr(sockfd));
  // FIXME poll with zero timeout to double confirm the new connection
  // FIXME use make_shared if necessary
  /*构建一个连接对象*/
  TcpConnectionPtr conn(new TcpConnection(loop_,
                                          connName,
                                          sockfd,
                                          localAddr,
                                          peerAddr));
  connections_[connName] = conn;
  conn->setConnectionCallback(connectionCallback_);
  conn->setMessageCallback(messageCallback_);
 
  conn->connectEstablished();
}
```

### TcpConnection头文件

TcpConnection.h

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
 
  // called when TcpServer accepts a new connection
  void connectEstablished();   // should be called only once
 
 private:
  enum StateE { /*kDisconnected, */kConnecting, kConnected/*, kDisconnecting*/ };
  void handleRead(Timestamp receiveTime);
  void setState(StateE s) { state_ = s; }
 
  EventLoop* loop_;         // 所属EventLoop
  string name_;             // 连接名
  /*连接状态*/
  StateE state_;  // FIXME: use atomic variable
  // we don't expose those classes to client.
  boost::scoped_ptr<Socket> socket_;
  boost::scoped_ptr<Channel> channel_;
  InetAddress localAddr_;
  InetAddress peerAddr_;
  /*连接套接字的回调函数*/
  ConnectionCallback connectionCallback_;
  /*消息被读到应用层后的回调函数*/
  MessageCallback messageCallback_;
};
 
typedef boost::shared_ptr<TcpConnection> TcpConnectionPtr;
 
}
}
 
#endif  // MUDUO_NET_TCPCONNECTION_H
```


### TcpConnection源文件

TcpConnection.cc

```c++
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
  //断言在eventloop IO线程中
  loop_->assertInLoopThread();
  //断言还没有连接
  assert(state_ == kConnecting);
  setState(kConnected);
  channel_->tie(shared_from_this());
  /*如果连接成功，则关注可读事件*/
  channel_->enableReading(); // TcpConnection所对应的通道加入到Poller关注
 
  connectionCallback_(shared_from_this());
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
  char buf[65536];
  ssize_t n = ::read(channel_->fd(), buf, sizeof buf);
  messageCallback_(shared_from_this(), buf, n);
}
```

### CallBack源文件

CallBack.cc

```c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_CALLBACKS_H
#define MUDUO_NET_CALLBACKS_H
 
#include <boost/function.hpp>
#include <boost/shared_ptr.hpp>
 
#include <muduo/base/Timestamp.h>
 
namespace muduo
{
/*
// Adapted from google-protobuf stubs/common.h
// see License in muduo/base/Types.h
template<typename To, typename From>
inline ::boost::shared_ptr<To> down_pointer_cast(const ::boost::shared_ptr<From>& f) {
  if (false) {
    implicit_cast<From*, To*>(0);
  }
 
#ifndef NDEBUG
  assert(f == NULL || dynamic_cast<To*>(get_pointer(f)) != NULL);
#endif
  return ::boost::static_pointer_cast<To>(f);
}*/
 
namespace net
{
 
// All client visible callbacks go here.
/*
class Buffer;
*/
class TcpConnection;
typedef boost::shared_ptr<TcpConnection> TcpConnectionPtr;
 
typedef boost::function<void()> TimerCallback;
 
typedef boost::function<void (const TcpConnectionPtr&)> ConnectionCallback;
/*typedef boost::function<void (const TcpConnectionPtr&)> CloseCallback;
typedef boost::function<void (const TcpConnectionPtr&)> WriteCompleteCallback;
typedef boost::function<void (const TcpConnectionPtr&, size_t)> HighWaterMarkCallback;
 
// the data has been read to (buf, len)
typedef boost::function<void (const TcpConnectionPtr&,
                              Buffer*,
                              Timestamp)> MessageCallback;
 
void defaultConnectionCallback(const TcpConnectionPtr& conn);
void defaultMessageCallback(const TcpConnectionPtr& conn,
                            Buffer* buffer,
                            Timestamp receiveTime);
                            */
typedef boost::function<void (const TcpConnectionPtr&,
                              const char* data,
                              ssize_t len)> MessageCallback;
 
}
}
 
#endif  // MUDUO_NET_CALLBACKS_H
```

### 测试程序1

```c++
#include <muduo/net/TcpServer.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
 
#include <stdio.h>
 
/*
 
这个程序主要用来测试TcpServer、TcpConnection
由于这里的TcpServer还没有实现关闭的事件处理，所以当对等放关闭时，服务器会一直处于高电平状态，
也就是说会一直触发
**/
using namespace muduo;
using namespace muduo::net;
 
void onConnection(const TcpConnectionPtr& conn)
{
  if (conn->connected())
  {
    printf("onConnection(): new connection [%s] from %s\n",
           conn->name().c_str(),
           conn->peerAddress().toIpPort().c_str());
  }
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
 
int main()
{
  printf("main(): pid = %d\n", getpid());
 
  InetAddress listenAddr(8888);
  EventLoop loop;
 
  TcpServer server(&loop, listenAddr, "TestServer");
  server.setConnectionCallback(onConnection);
  server.setMessageCallback(onMessage);
  server.start();
 
  loop.loop();
}
```

### 程序输出:

客户端

```c++
aaahuiahsdui crosoft Telnet Client
 
????????haracter is 'CTRL+]'
sadasd
 
asduhasuidhuiahsdiahsdjksdjfhsdiohahsoidoasudguagsdhjgashjdgajhksd
as
d
a
s
d
sad
 
Microsoft Telnet> quit
 
C:\Users\LaoHan>
```

### 服务器端

```c++
^Cubuntu@ubuntu-virtual-machine:~/pro/33/jmuduo$ ../build/debug/bin/reactor_test08
main(): pid = 6066
20131023 01:08:39.593827Z  6066 TRACE updateChannel fd = 4 events = 3 - EPollPoller.cc:104
20131023 01:08:39.594123Z  6066 TRACE EventLoop EventLoop created 0xBFBEBEDC in thread 6066 - EventLoop.cc:62
20131023 01:08:39.594172Z  6066 TRACE updateChannel fd = 5 events = 3 - EPollPoller.cc:104
20131023 01:08:39.594331Z  6066 TRACE updateChannel fd = 6 events = 3 - EPollPoller.cc:104
20131023 01:08:39.594387Z  6066 TRACE loop EventLoop 0xBFBEBEDC start looping - EventLoop.cc:94
20131023 01:08:43.065719Z  6066 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:43.066560Z  6066 TRACE printActiveChannels {6: IN }  - EventLoop.cc:257
20131023 01:08:43.066665Z  6066 INFO  TcpServer::newConnection [TestServer] - new connection [TestServer:0.0.0.0:8888#1] from 172.21.95.221:50007 - TcpServer.cc:74
20131023 01:08:43.066762Z  6066 DEBUG TcpConnection TcpConnection::ctor[TestServer:0.0.0.0:8888#1] at 0x9D80468 fd=8 - TcpConnection.cc:56
20131023 01:08:43.066802Z  6066 TRACE updateChannel fd = 8 events = 3 - EPollPoller.cc:104
onConnection(): new connection [TestServer:0.0.0.0:8888#1] from 172.21.95.221:50007
20131023 01:08:43.845782Z  6066 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:43.845853Z  6066 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257
onMessage(): received 1 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 01:08:44.122627Z  6066 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:44.122695Z  6066 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257
onMessage(): received 1 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 01:08:44.306479Z  6066 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:44.306545Z  6066 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257
onMessage(): received 1 bytes from connection [TestServer:0.0.0.0:8888#1]
onMessage(): received 0 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 01:08:23.429136Z  6063 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:23.429156Z  6063 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257
onMessage(): received 0 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 01:08:23.429186Z  6063 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:23.429205Z  6063 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257
onMessage(): received 0 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 01:08:23.429235Z  6063 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:23.429253Z  6063 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257
onMessage(): received 0 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 01:08:23.429283Z  6063 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:23.429302Z  6063 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257
onMessage(): received 0 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 01:08:23.429332Z  6063 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:23.429351Z  6063 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257
onMessage(): received 0 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 01:08:23.429381Z  6063 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:23.429399Z  6063 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257
onMessage(): received 0 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 01:08:23.429447Z  6063 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:23.429466Z  6063 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257
onMessage(): received 0 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 01:08:23.429497Z  6063 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 01:08:23.429515Z  6063 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257
onMessage(): received 0 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 01:08:23.429545Z  6063 TRACE poll 1 events happended - EPollPoller.cc:65
```

### 测试程序2

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
## [35-1] multiple reactors

<center> ![](https://i.loli.net/2019/05/03/5ccbe0857ddc3.png) </center>

- `muduo`库如何支持多线程
- `EventLoopThread`（`IO`线程类）
- `EventLoopThreadPool`（`IO`线程池类）
- `IO`线程池的功能是开启若干个`IO`线程，并让这些`IO`线程处于事件循环的状态

下面的这些代码可能和前面给出的源代码有些不一样，阅读的同学请注意了

### `EventLoopThreadPool`头文件

eventloopthreadpool.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is an internal header file, you should not include this.
 
#ifndef MUDUO_NET_EVENTLOOPTHREADPOOL_H
#define MUDUO_NET_EVENTLOOPTHREADPOOL_H
 
#include <muduo/base/Condition.h>
#include <muduo/base/Mutex.h>
 
#include <vector>
#include <boost/function.hpp>
#include <boost/noncopyable.hpp>
#include <boost/ptr_container/ptr_vector.hpp>
 
namespace muduo
{
 
namespace net
{
 
class EventLoop;
class EventLoopThread;
 
class EventLoopThreadPool : boost::noncopyable
{
 public:
  typedef boost::function<void(EventLoop*)> ThreadInitCallback;
 
  EventLoopThreadPool(EventLoop* baseLoop);
  ~EventLoopThreadPool();
  void setThreadNum(int numThreads) { numThreads_ = numThreads; }
  void start(const ThreadInitCallback& cb = ThreadInitCallback());
  EventLoop* getNextLoop();
 
 private:
 
  EventLoop* baseLoop_; // 与Acceptor所属EventLoop相同
  bool started_;        /*是否启动*/
  int numThreads_;      // 线程数
  int next_;            // 新连接到来，所选择的EventLoop对象下标
  boost::ptr_vector<EventLoopThread> threads_;        // IO线程列表
  std::vector<EventLoop*> loops_;                 // EventLoop列表
};
 
}
}
 
#endif  // MUDUO_NET_EVENTLOOPTHREADPOOL_H
```

### EventLoopThreadPool源文件

eventloopthreadpool.cc

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/net/EventLoopThreadPool.h>
 
#include <muduo/net/EventLoop.h>
#include <muduo/net/EventLoopThread.h>
 
#include <boost/bind.hpp>
 
using namespace muduo;
using namespace muduo::net;
 
 
EventLoopThreadPool::EventLoopThreadPool(EventLoop* baseLoop)
  : baseLoop_(baseLoop),  // 与Acceptor所属EventLoop相同
    started_(false),
    numThreads_(0),
    next_(0)
{
}
 
EventLoopThreadPool::~EventLoopThreadPool()
{
  // Don't delete loop, it's stack variable
}
/*每个线程初始化时的回调函数cb*/
void EventLoopThreadPool::start(const ThreadInitCallback& cb)
{
  assert(!started_);
  baseLoop_->assertInLoopThread();
 
  started_ = true;
 
  for (int i = 0; i < numThreads_; ++i)
  {
    EventLoopThread* t = new EventLoopThread(cb);
    threads_.push_back(t);
    loops_.push_back(t->startLoop());    // 启动EventLoopThread线程，在进入事件循环之前，会调用cb
  }
  if (numThreads_ == 0 && cb)
  {
    // 只有一个EventLoop，在这个EventLoop进入事件循环之前，调用cb
    cb(baseLoop_);
  }
}
 
EventLoop* EventLoopThreadPool::getNextLoop()
{
  baseLoop_->assertInLoopThread();
  EventLoop* loop = baseLoop_;
 
  // 如果loops_为空，则loop指向baseLoop_
  // 如果不为空，按照round-robin（RR，轮叫）的调度方式选择一个EventLoop
  if (!loops_.empty())
  {
    // round-robin
    loop = loops_[next_];
    ++next_;
    if (implicit_cast<size_t>(next_) >= loops_.size())
    {
      next_ = 0;
    }
  }
  return loop;
}
```

### TcpServer头文件

TcpServer.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_TCPSERVER_H
#define MUDUO_NET_TCPSERVER_H
 
#include <muduo/base/Types.h>
#include <muduo/net/TcpConnection.h>
 
#include <map>
#include <boost/noncopyable.hpp>
#include <boost/scoped_ptr.hpp>
 
namespace muduo
{
namespace net
{
 
class Acceptor;
class EventLoop;
class EventLoopThreadPool;
 
///
/// TCP server, supports single-threaded and thread-pool models.
///
/// This is an interface class, so don't expose too much details.
class TcpServer : boost::noncopyable
{
 public:
  typedef boost::function<void(EventLoop*)> ThreadInitCallback;
 
  //TcpServer(EventLoop* loop, const InetAddress& listenAddr);
  TcpServer(EventLoop* loop,
            const InetAddress& listenAddr,
            const string& nameArg);
  ~TcpServer();  // force out-line dtor, for scoped_ptr members.
 
  const string& hostport() const { return hostport_; }
  const string& name() const { return name_; }
 
  /// Set the number of threads for handling input.
  ///
  /// Always accepts new connection in loop's thread.
  /// Must be called before @c start
  /// @param numThreads
  /// - 0 means all I/O in loop's thread, no thread will created.
  ///   this is the default value.
  /// - 1 means all I/O in another thread.
  /// - N means a thread pool with N threads, new connections
  ///   are assigned on a round-robin basis.
  void setThreadNum(int numThreads);
  void setThreadInitCallback(const ThreadInitCallback& cb)
  { threadInitCallback_ = cb; }
 
  /// Starts the server if it's not listenning.
  ///
  /// It's harmless to call it multiple times.
  /// Thread safe.
  void start();
 
  /// Set connection callback.
  /// Not thread safe.
  // 设置连接到来或者连接关闭回调函数
  void setConnectionCallback(const ConnectionCallback& cb)
  { connectionCallback_ = cb; }
 
  /// Set message callback.
  /// Not thread safe.
  // 设置消息到来回调函数
  void setMessageCallback(const MessageCallback& cb)
  { messageCallback_ = cb; }
 
 
 private:
  /// Not thread safe, but in loop
  void newConnection(int sockfd, const InetAddress& peerAddr);
  /// Thread safe.
  void removeConnection(const TcpConnectionPtr& conn);
  /// Not thread safe, but in loop
  void removeConnectionInLoop(const TcpConnectionPtr& conn);
 
  typedef std::map<string, TcpConnectionPtr> ConnectionMap;
 
  EventLoop* loop_;  // the acceptor loop
  const string hostport_;       // 服务端口
  const string name_;           // 服务名
  boost::scoped_ptr<Acceptor> acceptor_; // avoid revealing Acceptor
  boost::scoped_ptr<EventLoopThreadPool> threadPool_; //线程池
  ConnectionCallback connectionCallback_;
  MessageCallback messageCallback_;
  ThreadInitCallback threadInitCallback_;   // IO线程池中的线程在进入事件循环前，会回调用此函数
  bool started_;
  // always in loop thread
  int nextConnId_;              // 下一个连接ID
  ConnectionMap connections_;   // 连接列表
};
 
}
}
 
#endif  // MUDUO_NET_TCPSERVER_H
```

### TcpServer源文件

TcpServer.cc

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/net/TcpServer.h>
 
#include <muduo/base/Logging.h>
#include <muduo/net/Acceptor.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/EventLoopThreadPool.h>
#include <muduo/net/SocketsOps.h>
 
#include <boost/bind.hpp>
 
#include <stdio.h>  // snprintf
 
using namespace muduo;
using namespace muduo::net;
 
TcpServer::TcpServer(EventLoop* loop, /*这是main reactor*/
                     const InetAddress& listenAddr,
                     const string& nameArg)
  : loop_(CHECK_NOTNULL(loop)),
    hostport_(listenAddr.toIpPort()),
    name_(nameArg),
    acceptor_(new Acceptor(loop, listenAddr)),
    threadPool_(new EventLoopThreadPool(loop)),
    /*connectionCallback_(defaultConnectionCallback),
    messageCallback_(defaultMessageCallback),*/
    started_(false),
    nextConnId_(1)
{
  // Acceptor::handleRead函数中会回调用TcpServer::newConnection
  // _1对应的是socket文件描述符，_2对应的是对等方的地址(InetAddress)
  acceptor_->setNewConnectionCallback(
      boost::bind(&TcpServer::newConnection, this, _1, _2));
}
 
TcpServer::~TcpServer()
{
  loop_->assertInLoopThread();
  LOG_TRACE << "TcpServer::~TcpServer [" << name_ << "] destructing";
 
  for (ConnectionMap::iterator it(connections_.begin());
      it != connections_.end(); ++it)
  {
    TcpConnectionPtr conn = it->second;
    it->second.reset();      // 释放当前所控制的对象，引用计数减一
    conn->getLoop()->runInLoop(
      boost::bind(&TcpConnection::connectDestroyed, conn));
    conn.reset();           // 释放当前所控制的对象，引用计数减一
  }
}
 
void TcpServer::setThreadNum(int numThreads)
{
  /*numThreads不包含main reactor thread*/
  assert(0 <= numThreads);
  threadPool_->setThreadNum(numThreads);
}
 
// 该函数多次调用是无害的
// 该函数可以跨线程调用
void TcpServer::start()
{
  if (!started_)
  {
    started_ = true;
    /*启动线程池*/
    threadPool_->start(threadInitCallback_);
  }
 
  if (!acceptor_->listenning())
  {
    // get_pointer返回原生指针
    loop_->runInLoop(
        boost::bind(&Acceptor::listen, get_pointer(acceptor_)));
  }
}
 
void TcpServer::newConnection(int sockfd, const InetAddress& peerAddr)
{
  loop_->assertInLoopThread();
  // 按照轮叫的方式选择一个EventLoop
  EventLoop* ioLoop = threadPool_->getNextLoop();
  char buf[32];
  snprintf(buf, sizeof buf, ":%s#%d", hostport_.c_str(), nextConnId_);
  ++nextConnId_;
  string connName = name_ + buf;
 
  LOG_INFO << "TcpServer::newConnection [" << name_
           << "] - new connection [" << connName
           << "] from " << peerAddr.toIpPort();
  InetAddress localAddr(sockets::getLocalAddr(sockfd));
  // FIXME poll with zero timeout to double confirm the new connection
  // FIXME use make_shared if necessary
  /*TcpConnectionPtr conn(new TcpConnection(loop_,
                                          connName,
                                          sockfd,
                                          localAddr,
                                          peerAddr));*/
 
  TcpConnectionPtr conn(new TcpConnection(ioLoop,
                                          connName,
                                          sockfd,
                                          localAddr,
                                          peerAddr));
 
  LOG_TRACE << "[1] usecount=" << conn.use_count();
  connections_[connName] = conn;
  LOG_TRACE << "[2] usecount=" << conn.use_count();
  conn->setConnectionCallback(connectionCallback_);
  conn->setMessageCallback(messageCallback_);
 
  conn->setCloseCallback(
      boost::bind(&TcpServer::removeConnection, this, _1));
 
  // conn->connectEstablished(); 这个表示直接在当前线程中调用
  ioLoop->runInLoop(boost::bind(&TcpConnection::connectEstablished, conn));
  LOG_TRACE << "[5] usecount=" << conn.use_count();
 
}
 
void TcpServer::removeConnection(const TcpConnectionPtr& conn)
{
    /*
  loop_->assertInLoopThread();
  LOG_INFO << "TcpServer::removeConnectionInLoop [" << name_
           << "] - connection " << conn->name();
 
 
  LOG_TRACE << "[8] usecount=" << conn.use_count();
  size_t n = connections_.erase(conn->name());
  LOG_TRACE << "[9] usecount=" << conn.use_count();
 
  (void)n;
  assert(n == 1);
 
  loop_->queueInLoop(
      boost::bind(&TcpConnection::connectDestroyed, conn));
  LOG_TRACE << "[10] usecount=" << conn.use_count();
  */
 
  loop_->runInLoop(boost::bind(&TcpServer::removeConnectionInLoop, this, conn));
 
}
 
void TcpServer::removeConnectionInLoop(const TcpConnectionPtr& conn)
{
  loop_->assertInLoopThread();
  LOG_INFO << "TcpServer::removeConnectionInLoop [" << name_
           << "] - connection " << conn->name();
 
 
  LOG_TRACE << "[8] usecount=" << conn.use_count();
  size_t n = connections_.erase(conn->name());
  LOG_TRACE << "[9] usecount=" << conn.use_count();
 
  (void)n;
  assert(n == 1);
 
  EventLoop* ioLoop = conn->getLoop();
  ioLoop->queueInLoop(
      boost::bind(&TcpConnection::connectDestroyed, conn));
 
  //loop_->queueInLoop(
  //    boost::bind(&TcpConnection::connectDestroyed, conn));
  LOG_TRACE << "[10] usecount=" << conn.use_count(); 
}
```

### 测试程序

``` c++
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
             const InetAddress& listenAddr, int numThreads)
    : loop_(loop),
      server_(loop, listenAddr, "TestServer"),
      numThreads_(numThreads)
  {
    server_.setConnectionCallback(
        boost::bind(&TestServer::onConnection, this, _1));
    server_.setMessageCallback(
        boost::bind(&TestServer::onMessage, this, _1, _2, _3));
    server_.setThreadNum(numThreads);
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
  int numThreads_;
};
 
 
int main()
{
  printf("main(): pid = %d\n", getpid());
 
  InetAddress listenAddr(8888);
  EventLoop loop;
 
  TestServer server(&loop, listenAddr,4);
  server.start();
 
  loop.loop();
}

   return n;
}
```
## [35-2] 文件描述符的分析

### code

``` c++
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

### 分析结果

``` c++
0 ， 1  ，2 进程默认打开
3 EpollerFd
4 timerFd
5 wakeupFd
6 listenFd
7 idleFd
 
ubuntu@ubuntu-virtual-machine:~/pro/35$ ./build/debug/bin/reactor_test09
main(): pid = 10797
20131023 08:05:04.423104Z 10797 TRACE updateChannel fd = 4 events = 3 - EPollPoller.cc:104
20131023 08:05:04.423413Z 10797 TRACE EventLoop EventLoop created 0xBF97AAB4 in thread 10797 - EventLoop.cc:62
20131023 08:05:04.423489Z 10797 TRACE updateChannel fd = 5 events = 3 - EPollPoller.cc:104
20131023 08:05:04.423883Z 10797 TRACE updateChannel fd = 6 events = 3 - EPollPoller.cc:104
20131023 08:05:04.423968Z 10797 TRACE loop EventLoop 0xBF97AAB4 start looping - EventLoop.cc:94
20131023 08:05:13.376959Z 10797 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 08:05:13.377558Z 10797 TRACE printActiveChannels {6: IN }  - EventLoop.cc:257-----》》有连接到来
20131023 08:05:13.377681Z 10797 INFO  TcpServer::newConnection [TestServer] - new connection [TestServer:0.0.0.0:8888#1] from 127.0.0.1:57756 - TcpServer.cc:93
20131023 08:05:13.377719Z 10797 DEBUG TcpConnection TcpConnection::ctor[TestServer:0.0.0.0:8888#1] at 0x8BB3490 fd=8 - TcpConnection.cc:62-------》》已连接的套接字
20131023 08:05:13.377746Z 10797 TRACE newConnection [1] usecount=1 - TcpServer.cc:111
20131023 08:05:13.377772Z 10797 TRACE newConnection [2] usecount=2 - TcpServer.cc:113
20131023 08:05:13.377819Z 10797 TRACE connectEstablished [3] usecount=6 - TcpConnection.cc:78
20131023 08:05:13.377846Z 10797 TRACE updateChannel fd = 8 events = 3 - EPollPoller.cc:104
onConnection(): new connection [TestServer:0.0.0.0:8888#1] from 127.0.0.1:57756
20131023 08:05:13.377902Z 10797 TRACE connectEstablished [4] usecount=6 - TcpConnection.cc:83
20131023 08:05:13.377915Z 10797 TRACE newConnection [5] usecount=2 - TcpServer.cc:122
20131023 08:05:23.388224Z 10797 TRACE poll  nothing happended - EPollPoller.cc:74
20131023 08:05:32.910851Z 10797 TRACE poll 1 events happended - EPollPoller.cc:65
20131023 08:05:32.910969Z 10797 TRACE printActiveChannels {8: IN }  - EventLoop.cc:257----->>已连接的套接字有可读事件
20131023 08:05:32.910990Z 10797 TRACE handleEvent [6] usecount=2 - Channel.cc:67
onMessage(): received 5 bytes from connection [TestServer:0.0.0.0:8888#1]
20131023 08:05:32.911066Z 10797 TRACE handleEvent [12] usecount=2 - Channel.cc:69
```

## [36-1] BUFFER

``` c++
1. 应用层缓冲区`Buffer`设计
2. 为什么需要有应用层缓冲区（详见`MuduoManual.pdf  P76`）
3. `Buffer`结构
4. `epoll`使用`LT`模式的原因
5. 与`poll`兼容
6. `LT`模式不会发生漏掉事件的`BUG`，但`POLLOUT`事件不能一开始就关注，否则会出现`busy loop`，而应该在`write`无法完全写入内核缓冲区的时候才关注，将未写入内核缓冲区的数据添加到应用层`output buffer`，直到应用层`output buffer`写完，停止关注`POLLOUT`事件。读写的时候不必等候`EAGAIN`，可以节省系统调用次数，降低延迟。（注：如果用`ET`模式，读的时候读到`EAGAIN`,写的时候直到`output buffer`写完或者`EAGAIN`）
```

### BUFFER头文件

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
/*
接收到数据，存至input buffer，通知上层的应用程序，OnMessage(Buffer *buf)回调，根据应用层协议
判断是否一个完整的包，codec，如果不是一条完整的消息，不会取走数据，也不会进行相应的处理，
如果是一条完整的信息，将取走这条信息，并进行相应的处理
 
对方发送了两条消息，50,350
接收方第一次接收了200，第二次接收了200
 
从socket读了 200 个字节，写到buffer中
从buffer中取回50 个字节
又从socket读了200个字节，写到buffer中
从buffer中取回350个字节，写到buffer中
从buffer中取回350个字节
 
*/
#ifndef MUDUO_NET_BUFFER_H
#define MUDUO_NET_BUFFER_H
 
#include <muduo/base/copyable.h>
#include <muduo/base/StringPiece.h>
#include <muduo/base/Types.h>
 
#include <muduo/net/Endian.h>
 
#include <algorithm>
#include <vector>
 
#include <assert.h>
#include <string.h>
//#include <unistd.h>  // ssize_t
 
namespace muduo
{
namespace net
{
 
/// A buffer class modeled after org.jboss.netty.buffer.ChannelBuffer
///
/// @code
/// +-------------------+------------------+------------------+
/// | prependable bytes |  readable bytes  |  writable bytes  |
/// |                   |     (CONTENT)    |                  |
/// +-------------------+------------------+------------------+
/// |                   |                  |                  |
/// 0      <=      readerIndex   <=   writerIndex    <=     size
/// @endcode
class Buffer : public muduo::copyable
{
 public:
  static const size_t kCheapPrepend = 8;
  static const size_t kInitialSize = 1024;
 
  Buffer()
  /*缓冲区的大小*/
    : buffer_(kCheapPrepend + kInitialSize),
    /*可读的开始位置*/
      readerIndex_(kCheapPrepend),
    /*可写的开始位置*/
      writerIndex_(kCheapPrepend)
  {
    assert(readableBytes() == 0);
    assert(writableBytes() == kInitialSize);
    assert(prependableBytes() == kCheapPrepend);
  }
 
  // default copy-ctor(拷贝构造函数), dtor and assignment are fine
 
  void swap(Buffer& rhs)
  {
    buffer_.swap(rhs.buffer_);
    std::swap(readerIndex_, rhs.readerIndex_);
    std::swap(writerIndex_, rhs.writerIndex_);
  }
    /*可读的字节数*/
  size_t readableBytes() const
  { return writerIndex_ - readerIndex_; }
    /*可写的字节数*/
  size_t writableBytes() const
  { return buffer_.size() - writerIndex_; }
    /*可预留的空间*/
  size_t prependableBytes() const
  { return readerIndex_; }
    /*可读位置指针*/
  const char* peek() const
  { return begin() + readerIndex_; }
 
  const char* findCRLF() const
  {
    const char* crlf = std::search(peek(), beginWrite(), kCRLF, kCRLF+2);
    return crlf == beginWrite() ? NULL : crlf;
  }
 
  const char* findCRLF(const char* start) const
  {
    assert(peek() <= start);
    assert(start <= beginWrite());
    const char* crlf = std::search(start, beginWrite(), kCRLF, kCRLF+2);
    return crlf == beginWrite() ? NULL : crlf;
  }
 
  // retrieve returns void, to prevent
  // string str(retrieve(readableBytes()), readableBytes());
  // the evaluation of two functions are unspecified
  /*取回数据*/
  void retrieve(size_t len)
  {
    assert(len <= readableBytes());
    if (len < readableBytes())
    {
      readerIndex_ += len;
    }
    else
    {
      retrieveAll();
    }
  }
  /*取回到指定位置的数据*/
  void retrieveUntil(const char* end)
  {
    assert(peek() <= end);
    assert(end <= beginWrite());
    retrieve(end - peek());
  }
  /*取回一个整数*/
  void retrieveInt32()
  {
    retrieve(sizeof(int32_t));
  }
  /*取回两个字节*/
  void retrieveInt16()
  {
    retrieve(sizeof(int16_t));
  }
  /*取回一个字节*/
  void retrieveInt8()
  {
    retrieve(sizeof(int8_t));
  }
  /*取回全部数据*/
  void retrieveAll()
  {
    readerIndex_ = kCheapPrepend;
    writerIndex_ = kCheapPrepend;
  }
 
  string retrieveAllAsString()
  {
    return retrieveAsString(readableBytes());;
  }
 
  string retrieveAsString(size_t len)
  {
    assert(len <= readableBytes());
    string result(peek(), len);
    retrieve(len);
    return result;
  }
 
  StringPiece toStringPiece() const
  {
    return StringPiece(peek(), static_cast<int>(readableBytes()));
  }
  /*追加数据*/
  void append(const StringPiece& str)
  {
    append(str.data(), str.size());
  }
 
  void append(const char* /*restrict*/ data, size_t len)
  {
    ensureWritableBytes(len);
    std::copy(data, data+len, beginWrite());
    hasWritten(len);
  }
 
  void append(const void* /*restrict*/ data, size_t len)
  {
    append(static_cast<const char*>(data), len);
  }
 
  // 确保缓冲区可写空间>=len，如果不足则扩充
  void ensureWritableBytes(size_t len)
  {
    if (writableBytes() < len)
    {
      makeSpace(len);
    }
    assert(writableBytes() >= len);
  }
  /*可写位置的指针*/
  char* beginWrite()
  { return begin() + writerIndex_; }
 
  const char* beginWrite() const
  { return begin() + writerIndex_; }
 
  void hasWritten(size_t len)
  { writerIndex_ += len; }
 
  ///
  /// Append int32_t using network endian
  /// 使用网络字节序追加到buffer
  void appendInt32(int32_t x)
  {
    int32_t be32 = sockets::hostToNetwork32(x);
    append(&be32, sizeof be32);
  }
 
  void appendInt16(int16_t x)
  {
    int16_t be16 = sockets::hostToNetwork16(x);
    append(&be16, sizeof be16);
  }
 
  void appendInt8(int8_t x)
  {
    append(&x, sizeof x);
  }
 
  ///
  /// Read int32_t from network endian
  ///
  /// Require: buf->readableBytes() >= sizeof(int32_t)
  int32_t readInt32()
  {
    int32_t result = peekInt32();
    retrieveInt32();
    return result;
  }
 
  int16_t readInt16()
  {
    int16_t result = peekInt16();
    retrieveInt16();
    return result;
  }
 
  int8_t readInt8()
  {
    int8_t result = peekInt8();
    retrieveInt8();
    return result;
  }
 
  ///
  /// Peek int32_t from network endian
  ///
  /// Require: buf->readableBytes() >= sizeof(int32_t)
  int32_t peekInt32() const
  {
    assert(readableBytes() >= sizeof(int32_t));
    int32_t be32 = 0;
    ::memcpy(&be32, peek(), sizeof be32);
    return sockets::networkToHost32(be32);
  }
 
  int16_t peekInt16() const
  {
    assert(readableBytes() >= sizeof(int16_t));
    int16_t be16 = 0;
    ::memcpy(&be16, peek(), sizeof be16);
    return sockets::networkToHost16(be16);
  }
 
  int8_t peekInt8() const
  {
    assert(readableBytes() >= sizeof(int8_t));
    int8_t x = *peek();
    return x;
  }
 
  ///
  /// Prepend int32_t using network endian
  ///
  void prependInt32(int32_t x)
  {
    int32_t be32 = sockets::hostToNetwork32(x);
    prepend(&be32, sizeof be32);
  }
 
  void prependInt16(int16_t x)
  {
    int16_t be16 = sockets::hostToNetwork16(x);
    prepend(&be16, sizeof be16);
  }
 
  void prependInt8(int8_t x)
  {
    prepend(&x, sizeof x);
  }
 
  void prepend(const void* /*restrict*/ data, size_t len)
  {
    assert(len <= prependableBytes());
    readerIndex_ -= len;
    const char* d = static_cast<const char*>(data);
    std::copy(d, d+len, begin()+readerIndex_);
  }
 
  // 收缩，保留reserve个字节
  void shrink(size_t reserve)
  {
    // FIXME: use vector::shrink_to_fit() in C++ 11 if possible.
    Buffer other;
    other.ensureWritableBytes(readableBytes()+reserve);
    other.append(toStringPiece());
    swap(other);
  }
 
  /// Read data directly into buffer.
  ///
  /// It may implement with readv(2)
  /// @return result of read(2), @c errno is saved
  ssize_t readFd(int fd, int* savedErrno);
 
 private:
 
  char* begin()
  { return &*buffer_.begin(); }
 
  const char* begin() const
  { return &*buffer_.begin(); }
 
  void makeSpace(size_t len)
  {
    if (writableBytes() + prependableBytes() < len + kCheapPrepend)
    {
      // FIXME: move readable data
      buffer_.resize(writerIndex_+len);
    }
    else
    {
      // move readable data to the front, make space inside buffer
      assert(kCheapPrepend < readerIndex_);
      size_t readable = readableBytes();
      std::copy(begin()+readerIndex_,
                begin()+writerIndex_,
                begin()+kCheapPrepend);
      readerIndex_ = kCheapPrepend;
      writerIndex_ = readerIndex_ + readable;
      assert(readable == readableBytes());
    }
  }
 
 private:
  std::vector<char> buffer_;  // vector用于替代固定大小数组
  size_t readerIndex_;          // 读位置
  size_t writerIndex_;          // 写位置
 
  static const char kCRLF[];    // "\r\n"
};
 
}
}
#endif  // MUDUO_NET_BUFFER_H
```

### BUFFER源文件

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
#include <muduo/net/Buffer.h>
 
#include <muduo/net/SocketsOps.h>
 
#include <errno.h>
#include <sys/uio.h>
 
using namespace muduo;
using namespace muduo::net;
 
const char Buffer::kCRLF[] = "\r\n";
 
const size_t Buffer::kCheapPrepend;
const size_t Buffer::kInitialSize;
 
// 结合栈上的空间，避免内存使用过大，提高内存使用率
// 如果有5K个连接，每个连接就分配64K+64K的缓冲区的话，将占用640M内存，
// 而大多数时候，这些缓冲区的使用率很低
ssize_t Buffer::readFd(int fd, int* savedErrno)
{
  // saved an ioctl()/FIONREAD call to tell how much to read
  // 节省一次ioctl系统调用（获取有多少可读数据）
  char extrabuf[65536];
  struct iovec vec[2];
  const size_t writable = writableBytes();
  // 第一块缓冲区
  vec[0].iov_base = begin()+writerIndex_;
  vec[0].iov_len = writable;
  // 第二块缓冲区
  vec[1].iov_base = extrabuf;
  vec[1].iov_len = sizeof extrabuf;
  const ssize_t n = sockets::readv(fd, vec, 2);
  if (n < 0)
  {
    *savedErrno = errno;
  }
  else if (implicit_cast<size_t>(n) <= writable)   //第一块缓冲区足够容纳
  {
    writerIndex_ += n;
  }
  else      // 当前缓冲区，不够容纳，因而数据被接收到了第二块缓冲区extrabuf，将其append至buffer
  {
    writerIndex_ = buffer_.size();
    append(extrabuf, n - writable);
  }
  // if (n == writable + sizeof extrabuf)
  // {
  //   goto line_30;
  // }
  return n;
}
```
## [37] BUFFER 

``` c++
应用程序想关闭连接，但是可能正处于发送数据的过程中，output buffer中有数据还没有发送完,不能直接调用`close`, `conn->send(buff)`==>`conn->shutdown()`==>`POLLOUT`事件.
关闭写的这一半: 连接状态更改为`kDisconnection`，并没有关闭连接
服务器主动断开与客户端的连接, 这意味着 客户端 `read`返回`0`, `close(conn)` ==> `POLLHUP|POLLIN`
如果是客户端主动关闭连接，那么服务器端只返回`POLLIN`,如果是服务器端主动关闭，然后客户端再关闭，那么服务器端返回`POLLIN`和`POLLHUP`事件
```
### 测试程序

服务器

``` c++
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
 
    message1_.resize(100);
    message2_.resize(200);
    std::fill(message1_.begin(), message1_.end(), 'A');
    std::fill(message2_.begin(), message2_.end(), 'B');
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
      conn->send(message1_);
      conn->send(message2_);
      conn->shutdown();
    }
    else
    {
      printf("onConnection(): connection [%s] is down\n",
             conn->name().c_str());
    }
  }
 
  void onMessage(const TcpConnectionPtr& conn,
                 Buffer* buf,
                 Timestamp receiveTime)
  {
    muduo::string msg(buf->retrieveAllAsString());
    printf("onMessage(): received %zd bytes from connection [%s] at %s\n",
           msg.size(),
           conn->name().c_str(),
           receiveTime.toFormattedString().c_str());
 
    conn->send(msg);
  }
 
  EventLoop* loop_;
  TcpServer server_;
 
  muduo::string message1_;
  muduo::string message2_;
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

### 客户端

``` c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
 
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
 //该程序可能出现粘包问题
#define ERR_EXIT(m) \
        do \
        { \
                perror(m); \
                exit(EXIT_FAILURE); \
        } while(0)
 
int main(void)
{
    int sock;
    if ((sock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
        ERR_EXIT("socket");
 
    struct sockaddr_in servaddr;
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(8888);
    servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
 
    if (connect(sock, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
        ERR_EXIT("connect");
 
    char buf[1024] ={0};
    while (1)
    {
        int ret = read(sock, buf, sizeof(buf));
        printf("ret=%d\n", ret);
        if (ret == 0)
            break;
 
        fputs(buf, stdout);
        fflush(stdout);
        memset(buf, 0, sizeof(buf));
    }
 
    sleep(10);
    close(sock);
 
    return 0;
}

```

### 程序输出

服务器：

``` c++
ubuntu@ubuntu-virtual-machine:~/pro/37$ ./build/debug/bin/reactor_test12
main(): pid = 8683
20131024 07:19:37.457271Z  8683 TRACE updateChannel fd = 4 events = 3 - EPollPoller.cc:104
20131024 07:19:37.457470Z  8683 TRACE EventLoop EventLoop created 0xBFDDAB44 in thread 8683 - EventLoop.cc:62
20131024 07:19:37.457485Z  8683 TRACE updateChannel fd = 5 events = 3 - EPollPoller.cc:104
20131024 07:19:37.457645Z  8683 TRACE updateChannel fd = 6 events = 3 - EPollPoller.cc:104
20131024 07:19:37.457661Z  8683 TRACE loop EventLoop 0xBFDDAB44 start looping - EventLoop.cc:94
20131024 07:19:47.468016Z  8683 TRACE poll  nothing happended - EPollPoller.cc:74
20131024 07:19:57.478500Z  8683 TRACE poll  nothing happended - EPollPoller.cc:74
20131024 07:20:07.488977Z  8683 TRACE poll  nothing happended - EPollPoller.cc:74
20131024 07:20:17.499503Z  8683 TRACE poll  nothing happended - EPollPoller.cc:74
20131024 07:20:27.509985Z  8683 TRACE poll  nothing happended - EPollPoller.cc:74
20131024 07:20:37.520451Z  8683 TRACE poll  nothing happended - EPollPoller.cc:74
20131024 07:20:47.528439Z  8683 TRACE poll  nothing happended - EPollPoller.cc:74
20131024 07:20:57.538942Z  8683 TRACE poll  nothing happended - EPollPoller.cc:74
20131024 07:21:01.262358Z  8683 TRACE poll 1 events happended - EPollPoller.cc:65
20131024 07:21:01.262792Z  8683 TRACE printActiveChannels {6: IN }  - EventLoop.cc:257
20131024 07:21:01.262820Z  8683 TRACE handleEventWithGuard 2222222222222222POLLIN22222222222222222222 - Channel.cc:89
20131024 07:21:01.262881Z  8683 INFO  TcpServer::newConnection [TestServer] - new connection [TestServer:0.0.0.0:8888#1] from 127.0.0.1:56659 - TcpServer.cc:93
20131024 07:21:01.262916Z  8683 DEBUG TcpConnection TcpConnection::ctor[TestServer:0.0.0.0:8888#1] at 0x96C65D0 fd=8 - TcpConnection.cc:62
20131024 07:21:01.262934Z  8683 TRACE newConnection [1] usecount=1 - TcpServer.cc:111
20131024 07:21:01.262961Z  8683 TRACE newConnection [2] usecount=2 - TcpServer.cc:113
20131024 07:21:01.262981Z  8683 TRACE connectEstablished [3] usecount=6 - TcpConnection.cc:231
20131024 07:21:01.262998Z  8683 TRACE updateChannel fd = 8 events = 3 - EPollPoller.cc:104
onConnection(): new connection [TestServer:0.0.0.0:8888#1] from 127.0.0.1:56659
20131024 07:21:01.263363Z  8683 TRACE connectEstablished [4] usecount=6 - TcpConnection.cc:236
20131024 07:21:01.263378Z  8683 TRACE newConnection [5] usecount=2 - TcpServer.cc:122
20131024 07:21:11.265287Z  8683 TRACE poll 1 events happended - EPollPoller.cc:65
20131024 07:21:11.265469Z  8683 TRACE printActiveChannels {8: IN HUP }  - EventLoop.cc:257
20131024 07:21:11.265483Z  8683 TRACE handleEvent [6] usecount=2 - Channel.cc:67
20131024 07:21:11.265491Z  8683 TRACE handleEventWithGuard 1111111111111111POLLHUP1111111111111111111 - Channel.cc:85
20131024 07:21:11.265498Z  8683 TRACE handleEventWithGuard 2222222222222222POLLIN22222222222222222222 - Channel.cc:89
20131024 07:21:11.265556Z  8683 TRACE handleClose fd = 8 state = 3 - TcpConnection.cc:297
20131024 07:21:11.265567Z  8683 TRACE updateChannel fd = 8 events = 0 - EPollPoller.cc:104
onConnection(): connection [TestServer:0.0.0.0:8888#1] is down
20131024 07:21:11.265588Z  8683 TRACE handleClose [7] usecount=3 - TcpConnection.cc:305
20131024 07:21:11.265604Z  8683 INFO  TcpServer::removeConnectionInLoop [TestServer] - connection TestServer:0.0.0.0:8888#1 - TcpServer.cc:153
20131024 07:21:11.265611Z  8683 TRACE removeConnectionInLoop [8] usecount=6 - TcpServer.cc:157
20131024 07:21:11.265626Z  8683 TRACE removeConnectionInLoop [9] usecount=5 - TcpServer.cc:159
20131024 07:21:11.265639Z  8683 TRACE removeConnectionInLoop [10] usecount=6 - TcpServer.cc:170
20131024 07:21:11.265646Z  8683 TRACE handleClose [11] usecount=3 - TcpConnection.cc:308
20131024 07:21:11.265652Z  8683 TRACE handleEvent [12] usecount=2 - Channel.cc:69
20131024 07:21:11.265659Z  8683 TRACE removeChannel fd = 8 - EPollPoller.cc:147
20131024 07:21:11.265682Z  8683 DEBUG ~TcpConnection TcpConnection::dtor[TestServer:0.0.0.0:8888#1] at 0x96C65D0 fd=8 - TcpConnection.cc:69
20131024 07:21:21.275816Z  8683 TRACE poll  nothing happended - EPollPoller.cc:74
20131024 07:21:31.285830Z  8683 TRACE poll  nothing happended - EPollPoller.cc:74
20131024 07:21:41.296314Z  8683 TRACE poll  nothing happended - EPollPoller.cc:74
^C
ubuntu@ubuntu-virtual-machine:~/pro/37$
```

客户端：

``` c++
ubuntu@ubuntu-virtual-machine:~/pro/37$ ./a.out
ret=300
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBret=0
```
## [38] 完善TcpConnection

完善`TcpConnection`
`WriteCompleteCallback`含义
`HighWaterMarkCallback`含义
`boost::any context_`
`signal(SIGPIPE, SIG_IGN)` // 请看`channel class`
可变类型解决方案
`void*.` 这种方法不是类型安全的
`boost::any`
`boost::any`
任意类型的类型安全存储以及安全的取回
在标准库容器中存放不同类型的方法，比如说`vector<boost::any>`
`TcpConnection` 完整的头文件

TcpConnection.h

``` c++
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
#include <muduo/net/Buffer.h>
#include <muduo/net/InetAddress.h>
 
#include <boost/any.hpp>
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
 
  // void send(string&& message); // C++11
  void send(const void* message, size_t len);
  void send(const StringPiece& message);
  // void send(Buffer&& message); // C++11
  void send(Buffer* message);  // this one will swap data
  void shutdown(); // NOT thread safe, no simultaneous calling
  void setTcpNoDelay(bool on);
 
  void setContext(const boost::any& context)
  { context_ = context; }
 
  const boost::any& getContext() const
  { return context_; }
 
  boost::any* getMutableContext()
  { return &context_; }
 
  void setConnectionCallback(const ConnectionCallback& cb)
  { connectionCallback_ = cb; }
 
  void setMessageCallback(const MessageCallback& cb)
  { messageCallback_ = cb; }
 
  void setWriteCompleteCallback(const WriteCompleteCallback& cb)
  { writeCompleteCallback_ = cb; }
 
  void setHighWaterMarkCallback(const HighWaterMarkCallback& cb, size_t highWaterMark)
  { highWaterMarkCallback_ = cb; highWaterMark_ = highWaterMark; }
 
  Buffer* inputBuffer()
  { return &inputBuffer_; }
 
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
  void handleWrite();
  void handleClose();
  void handleError();
  void sendInLoop(const StringPiece& message);
  void sendInLoop(const void* message, size_t len);
  void shutdownInLoop();
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
  WriteCompleteCallback writeCompleteCallback_;     // 数据发送完毕回调函数，即所有的用户数据都已拷贝到内核缓冲区时回调该函数
                                                    // outputBuffer_被清空也会回调该函数，可以理解为低水位标回调函数
  HighWaterMarkCallback highWaterMarkCallback_;     // 高水位标回调函数
  CloseCallback closeCallback_;
  size_t highWaterMark_;        // 高水位标
  Buffer inputBuffer_;          // 应用层接收缓冲区
  Buffer outputBuffer_;         // 应用层发送缓冲区
  boost::any context_;          // 绑定一个未知类型的上下文对象
};
 
typedef boost::shared_ptr<TcpConnection> TcpConnectionPtr;
 
}
}
 
#endif  // MUDUO_NET_TCPCONNECTION_H
```

### TcpConnection 完整的源文件

TcpConnection.cc

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
    peerAddr_(peerAddr),
    highWaterMark_(64*1024*1024)
{
  // 通道可读事件到来的时候，回调TcpConnection::handleRead，_1是事件发生时间
  channel_->setReadCallback(
      boost::bind(&TcpConnection::handleRead, this, _1));
  // 通道可写事件到来的时候，回调TcpConnection::handleWrite
  channel_->setWriteCallback(
      boost::bind(&TcpConnection::handleWrite, this));
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
 
// 线程安全，可以跨线程调用
void TcpConnection::send(const void* data, size_t len)
{
  if (state_ == kConnected)
  {
    if (loop_->isInLoopThread())
    {
      sendInLoop(data, len);
    }
    else
    {
      string message(static_cast<const char*>(data), len);
      loop_->runInLoop(
          boost::bind(&TcpConnection::sendInLoop,
                      this,
                      message));
    }
  }
}
 
// 线程安全，可以跨线程调用
void TcpConnection::send(const StringPiece& message)
{
  if (state_ == kConnected)
  {
    if (loop_->isInLoopThread())
    {
      sendInLoop(message);
    }
    else
    {
      loop_->runInLoop(
          boost::bind(&TcpConnection::sendInLoop,
                      this,
                      message.as_string()));
                    //std::forward<string>(message)));
    }
  }
}
 
// 线程安全，可以跨线程调用
void TcpConnection::send(Buffer* buf)
{
  if (state_ == kConnected)
  {
    if (loop_->isInLoopThread())
    {
      sendInLoop(buf->peek(), buf->readableBytes());
      buf->retrieveAll();
    }
    else
    {
      loop_->runInLoop(
          boost::bind(&TcpConnection::sendInLoop,
                      this,
                      buf->retrieveAllAsString()));
                    //std::forward<string>(message)));
    }
  }
}
 
void TcpConnection::sendInLoop(const StringPiece& message)
{
  sendInLoop(message.data(), message.size());
}
 
void TcpConnection::sendInLoop(const void* data, size_t len)
{
  /*
  loop_->assertInLoopThread();
  sockets::write(channel_->fd(), data, len);
  */
 
  loop_->assertInLoopThread();
  ssize_t nwrote = 0;
  size_t remaining = len;
  bool error = false;
  if (state_ == kDisconnected)
  {
    LOG_WARN << "disconnected, give up writing";
    return;
  }
  // if no thing in output queue, try writing directly
  // 通道没有关注可写事件并且发送缓冲区没有数据，直接write
  if (!channel_->isWriting() && outputBuffer_.readableBytes() == 0)
  {
    nwrote = sockets::write(channel_->fd(), data, len);
    if (nwrote >= 0)
    {
      remaining = len - nwrote;
      // 写完了，回调writeCompleteCallback_
      if (remaining == 0 && writeCompleteCallback_)
      {
        loop_->queueInLoop(boost::bind(writeCompleteCallback_, shared_from_this()));
      }
    }
    else // nwrote < 0
    {
      nwrote = 0;
      if (errno != EWOULDBLOCK)
      {
        LOG_SYSERR << "TcpConnection::sendInLoop";
        if (errno == EPIPE) // FIXME: any others?
        {
          error = true;
        }
      }
    }
  }
 
  assert(remaining <= len);
  // 没有错误，并且还有未写完的数据（说明内核发送缓冲区满，要将未写完的数据添加到output buffer中）
  if (!error && remaining > 0)
  {
    LOG_TRACE << "I am going to write more data";
    size_t oldLen = outputBuffer_.readableBytes();
    // 如果超过highWaterMark_（高水位标），回调highWaterMarkCallback_
    //即使oldLen + remaining >= highWaterMark_ ，remain的数据也是可以存放到buffer中的，因为buffer
    //自动伸缩的
    if (oldLen + remaining >= highWaterMark_
        && oldLen < highWaterMark_
        && highWaterMarkCallback_)
    {
      loop_->queueInLoop(boost::bind(highWaterMarkCallback_, shared_from_this(), oldLen + remaining));
    }
    outputBuffer_.append(static_cast<const char*>(data)+nwrote, remaining);
    if (!channel_->isWriting())
    {
      channel_->enableWriting();     // 关注POLLOUT事件
    }
  }
}
 
void TcpConnection::shutdown()
{
  // FIXME: use compare and swap
  if (state_ == kConnected)
  {
    setState(kDisconnecting);
    // FIXME: shared_from_this()?
    loop_->runInLoop(boost::bind(&TcpConnection::shutdownInLoop, this));
  }
}
 
void TcpConnection::shutdownInLoop()
{
  loop_->assertInLoopThread();
  if (!channel_->isWriting())
  {
    // we are not writing
    socket_->shutdownWrite();
  }
}
 
void TcpConnection::setTcpNoDelay(bool on)
{
  socket_->setTcpNoDelay(on);
}
 
void TcpConnection::connectEstablished()
{
  loop_->assertInLoopThread();
  assert(state_ == kConnecting);
  setState(kConnected);
  LOG_TRACE << "[3] usecount=" << shared_from_this().use_count();
  channel_->tie(shared_from_this());
  channel_->enableReading(); // TcpConnection所对应的通道加入到Poller关注
 
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
 
  /*
  loop_->assertInLoopThread();
  int savedErrno = 0;
  char buf[65536];
  ssize_t n = ::read(channel_->fd(), buf, sizeof buf);
  if (n > 0)
  {
    messageCallback_(shared_from_this(), buf, n);
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
  /*一次性读完内核缓冲区的数据，这可以防止busy loop*/
  ssize_t n = inputBuffer_.readFd(channel_->fd(), &savedErrno);
  if (n > 0)
  {
    messageCallback_(shared_from_this(), &inputBuffer_, receiveTime);
  }
  else if (n == 0)
  {
    //读到文件eof ，就是对等方已经关闭连接
    handleClose();
  }
  else
  {
    errno = savedErrno;
    LOG_SYSERR << "TcpConnection::handleRead";
    handleError();
  }
}
 
// 内核发送缓冲区有空间了，回调该函数
void TcpConnection::handleWrite()
{
  loop_->assertInLoopThread();
  /*通道有关注write事件 */
  if (channel_->isWriting())
  {
    /*写数据到内核中*/
    ssize_t n = sockets::write(channel_->fd(),
                               outputBuffer_.peek(),
                               outputBuffer_.readableBytes());
    if (n > 0)
    {
      outputBuffer_.retrieve(n);
      if (outputBuffer_.readableBytes() == 0)    // 发送缓冲区已清空
      {
        channel_->disableWriting();      // 停止关注POLLOUT事件，以免出现busy loop
        if (writeCompleteCallback_)     // 回调writeCompleteCallback_
        {
          // 应用层发送缓冲区被清空，就回调用writeCompleteCallback_
          loop_->queueInLoop(boost::bind(writeCompleteCallback_, shared_from_this()));
        }
        /*如果状态码是kDisconnecting ，说明应用层已经发起shutdown命令了，这是最后一个数据包，
        所以发完就可以关闭了。
        请参照void TcpConnection::shutdownInLoop()
        */
        if (state_ == kDisconnecting)   // 发送缓冲区已清空并且连接状态是kDisconnecting, 要关闭连接
        {
          shutdownInLoop();     // 关闭连接
        }
      }
      else
      {
        LOG_TRACE << "I am going to write more data";
      }
    }
    else
    {
      LOG_SYSERR << "TcpConnection::handleWrite";
      // if (state_ == kDisconnecting)
      // {
      //   shutdownInLoop();
      // }
    }
  }
  else
  {
    LOG_TRACE << "Connection fd = " << channel_->fd()
              << " is down, no more writing";
  }
}
 
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

### 测试程序

``` c++
#include <muduo/net/TcpServer.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
 
#include <boost/bind.hpp>
 
#include <stdio.h>
/*
程序说明：
  客户端请求连接-------》服务器马上发送一堆数据
 
*/
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
    server_.setWriteCompleteCallback(
      boost::bind(&TestServer::onWriteComplete, this, _1));
 
    // 这是一个数据生成协议
    string line;
    for (int i = 33; i < 127; ++i)
    {
      line.push_back(char(i));
    }
    line += line;
 
    for (size_t i = 0; i < 127-33; ++i)
    {
      message_ += line.substr(i, 72) + '\n';
    }
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
 
      conn->setTcpNoDelay(true);
      /*连接到来直接发送数据*/
      conn->send(message_);
    }
    else
    {
      printf("onConnection(): connection [%s] is down\n",
             conn->name().c_str());
    }
  }
 
  void onMessage(const TcpConnectionPtr& conn,
                 Buffer* buf,
                 Timestamp receiveTime)
  {
    muduo::string msg(buf->retrieveAllAsString());
    printf("onMessage(): received %zd bytes from connection [%s] at %s\n",
           msg.size(),
           conn->name().c_str(),
           receiveTime.toFormattedString().c_str());
 
    conn->send(msg);
  }
 
  void onWriteComplete(const TcpConnectionPtr& conn)
  { /*可写事件，在发送*/
    conn->send(message_);
  }
 
  EventLoop* loop_;
  TcpServer server_;
 
  muduo::string message_;
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

## [39] TcpClient和connector

- `muduo`库对编写`tcp`(主动发起连接`TcpClient`)
- 客户端程序的支持`Connector`(包含了一个`Connector`对象)

### 测试程序

- Reactor_test11.cc    
// echo server
- TcpClient_test.cc    
// echo client

### TcpClient 头文件
TcpClient.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_TCPCLIENT_H
#define MUDUO_NET_TCPCLIENT_H
 
#include <boost/noncopyable.hpp>
 
#include <muduo/base/Mutex.h>
#include <muduo/net/TcpConnection.h>
 
namespace muduo
{
namespace net
{
 
class Connector;
typedef boost::shared_ptr<Connector> ConnectorPtr;
 
class TcpClient : boost::noncopyable
{
 public:
  // TcpClient(EventLoop* loop);
  // TcpClient(EventLoop* loop, const string& host, uint16_t port);
  TcpClient(EventLoop* loop,
            const InetAddress& serverAddr,
            const string& name);
  ~TcpClient();  // force out-line dtor, for scoped_ptr members.
 
  void connect();
  void disconnect();
  void stop();
 
  TcpConnectionPtr connection() const
  {
    MutexLockGuard lock(mutex_);
    return connection_;
  }
 
  bool retry() const;
  void enableRetry() { retry_ = true; }
 
  /// Set connection callback.
  /// Not thread safe.
  void setConnectionCallback(const ConnectionCallback& cb)
  { connectionCallback_ = cb; }
 
  /// Set message callback.
  /// Not thread safe.
  void setMessageCallback(const MessageCallback& cb)
  { messageCallback_ = cb; }
 
  /// Set write complete callback.
  /// Not thread safe.
  void setWriteCompleteCallback(const WriteCompleteCallback& cb)
  { writeCompleteCallback_ = cb; }
 
 private:
  /// Not thread safe, but in loop
  void newConnection(int sockfd);
  /// Not thread safe, but in loop
  void removeConnection(const TcpConnectionPtr& conn);
 
  EventLoop* loop_;
  ConnectorPtr connector_;  // 用于主动发起连接
  const string name_;       // 名称
  ConnectionCallback connectionCallback_;       // 连接建立回调函数
  MessageCallback messageCallback_;             // 消息到来回调函数
  WriteCompleteCallback writeCompleteCallback_; // 数据发送完毕回调函数
  bool retry_;   // 重连，是指连接建立之后又断开的时候是否重连，
                //这个跟connector里面的retry是不一样的，connector里面的retry表示连接不成功时是否要重连
  bool connect_; // atomic 是否发起连接
  // always in loop thread
  int nextConnId_;          // name_ + nextConnId_用于标识一个连接
  mutable MutexLock mutex_;
  TcpConnectionPtr connection_; // Connector连接成功以后，得到一个TcpConnection
};
 
}
}
 
#endif  // MUDUO_NET_TCPCLIENT_H
```
### TcpClient 源文件

TcpClient.cc

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
 
#include <muduo/net/TcpClient.h>
 
#include <muduo/base/Logging.h>
#include <muduo/net/Connector.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/SocketsOps.h>
 
#include <boost/bind.hpp>
 
#include <stdio.h>  // snprintf
 
using namespace muduo;
using namespace muduo::net;
 
// TcpClient::TcpClient(EventLoop* loop)
//   : loop_(loop)
// {
// }
 
// TcpClient::TcpClient(EventLoop* loop, const string& host, uint16_t port)
//   : loop_(CHECK_NOTNULL(loop)),
//     serverAddr_(host, port)
// {
// }
 
namespace muduo
{
namespace net
{
namespace detail
{
 
void removeConnection(EventLoop* loop, const TcpConnectionPtr& conn)
{
  loop->queueInLoop(boost::bind(&TcpConnection::connectDestroyed, conn));
}
 
void removeConnector(const ConnectorPtr& connector)
{
  //connector->
}
 
}
}
}
 
TcpClient::TcpClient(EventLoop* loop,
                     const InetAddress& serverAddr,
                     const string& name)
  : loop_(CHECK_NOTNULL(loop)),
    connector_(new Connector(loop, serverAddr)),
    name_(name),
    connectionCallback_(defaultConnectionCallback),
    messageCallback_(defaultMessageCallback),
    retry_(false),
    connect_(true),
    nextConnId_(1)
{
  // 设置连接成功回调函数
  connector_->setNewConnectionCallback(
      boost::bind(&TcpClient::newConnection, this, _1));
  // FIXME setConnectFailedCallback
  LOG_INFO << "TcpClient::TcpClient[" << name_
           << "] - connector " << get_pointer(connector_);
}
 
TcpClient::~TcpClient()
{
  LOG_INFO << "TcpClient::~TcpClient[" << name_
           << "] - connector " << get_pointer(connector_);
  TcpConnectionPtr conn;
  {
    MutexLockGuard lock(mutex_);
    conn = connection_;
  }
  if (conn)
  {
    // FIXME: not 100% safe, if we are in different thread
 
    // 重新设置TcpConnection中的closeCallback_为detail::removeConnection
    //这里不用TcpClient::removeConnection的原因是 TcpClient::removeConnection 有重连功能,
    //这里已经不需要重连了，直接调用detail::removeConnection就行了
    CloseCallback cb = boost::bind(&detail::removeConnection, loop_, _1);
    loop_->runInLoop(
        boost::bind(&TcpConnection::setCloseCallback, conn, cb));
  }
  else
  {
    // 这种情况，说明connector处于未连接状态，将connector_停止
    connector_->stop();
    // FIXME: HACK
    loop_->runAfter(1, boost::bind(&detail::removeConnector, connector_));
  }
}
 
void TcpClient::connect()
{
  // FIXME: check state
  LOG_INFO << "TcpClient::connect[" << name_ << "] - connecting to "
           << connector_->serverAddress().toIpPort();
  connect_ = true;
  connector_->start();   // 发起连接
}
 
// 用于连接已建立的情况下，关闭连接
void TcpClient::disconnect()
{
  connect_ = false;
 
  {
    MutexLockGuard lock(mutex_);
    if (connection_)
    {
      connection_->shutdown();
    }
  }
}
 
// 停止connector_ ,连接尚未成功时进行断开
void TcpClient::stop()
{
  connect_ = false;
  connector_->stop();
}
 
//有新的连接到来
void TcpClient::newConnection(int sockfd)
{
  loop_->assertInLoopThread();
  InetAddress peerAddr(sockets::getPeerAddr(sockfd));
  char buf[32];
  snprintf(buf, sizeof buf, ":%s#%d", peerAddr.toIpPort().c_str(), nextConnId_);
  ++nextConnId_;
  string connName = name_ + buf;
 
  InetAddress localAddr(sockets::getLocalAddr(sockfd));
  // FIXME poll with zero timeout to double confirm the new connection
  // FIXME use make_shared if necessary
  TcpConnectionPtr conn(new TcpConnection(loop_,
                                          connName,
                                          sockfd,
                                          localAddr,
                                          peerAddr));
 
  conn->setConnectionCallback(connectionCallback_);
  conn->setMessageCallback(messageCallback_);
  conn->setWriteCompleteCallback(writeCompleteCallback_);
  conn->setCloseCallback(
      boost::bind(&TcpClient::removeConnection, this, _1)); // FIXME: unsafe
  {
    MutexLockGuard lock(mutex_);
    connection_ = conn;     // 保存TcpConnection
  }
  conn->connectEstablished();        // 这里回调connectionCallback_
}
 
/*移除connector*/
void TcpClient::removeConnection(const TcpConnectionPtr& conn)
{
  loop_->assertInLoopThread();
  assert(loop_ == conn->getLoop());
 
  {
    MutexLockGuard lock(mutex_);
    assert(connection_ == conn);
    connection_.reset();
  }
  /*放到IO线程中销毁*/
  loop_->queueInLoop(boost::bind(&TcpConnection::connectDestroyed, conn));
  //如果需要重连，则重新连接
  if (retry_ && connect_)
  {
    LOG_INFO << "TcpClient::connect[" << name_ << "] - Reconnecting to "
             << connector_->serverAddress().toIpPort();
    // 这里的重连是指连接建立成功之后被断开的重连
    connector_->restart();
  }
}
```

### Connector 头文件

Connector.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is an internal header file, you should not include this.
 
#ifndef MUDUO_NET_CONNECTOR_H
#define MUDUO_NET_CONNECTOR_H
 
#include <muduo/net/InetAddress.h>
 
#include <boost/enable_shared_from_this.hpp>
#include <boost/function.hpp>
#include <boost/noncopyable.hpp>
#include <boost/scoped_ptr.hpp>
 
namespace muduo
{
namespace net
{
 
class Channel;
class EventLoop;
 
// 主动发起连接，带有自动重连功能
class Connector : boost::noncopyable,
                  public boost::enable_shared_from_this<Connector>
{
 public:
  typedef boost::function<void (int sockfd)> NewConnectionCallback;
 
  Connector(EventLoop* loop, const InetAddress& serverAddr);
  ~Connector();
  /*设置连接成功后的回调函数*/
  void setNewConnectionCallback(const NewConnectionCallback& cb)
  { newConnectionCallback_ = cb; }
 
  void start();  // can be called in any thread
  void restart();  // must be called in loop thread
  void stop();  // can be called in any thread
//服务器IP地址
  const InetAddress& serverAddress() const { return serverAddr_; }
 
 private:
  enum States { kDisconnected/*断开状态*/, kConnecting/*正在连接中*/, kConnected/*已连接*/ };
  static const int kMaxRetryDelayMs = 30*1000;          // 30秒，最大重连延迟时间
  static const int kInitRetryDelayMs = 500;             // 0.5秒，初始状态，连接不上，0.5秒后重连
 
  void setState(States s) { state_ = s; }
  void startInLoop();
  void stopInLoop();
  void connect();
  void connecting(int sockfd);
  void handleWrite();
  void handleError();
  void retry(int sockfd);
  int removeAndResetChannel();
  void resetChannel();
 
  EventLoop* loop_;         // 所属EventLoop
  InetAddress serverAddr_;  // 服务器端地址
  bool connect_; // atomic是否连接
  States state_;  // FIXME: use atomic variable
  boost::scoped_ptr<Channel> channel_;    // Connector所对应的Channel
  NewConnectionCallback newConnectionCallback_;     // 连接成功回调函数，
  int retryDelayMs_;        // 重连延迟时间（单位：毫秒）
};
 
}
}
 
#endif  // MUDUO_NET_CONNECTOR_H
```

### Connector 源文件

Connector.cc

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
 
#include <muduo/net/Connector.h>
 
#include <muduo/base/Logging.h>
#include <muduo/net/Channel.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/SocketsOps.h>
 
#include <boost/bind.hpp>
 
#include <errno.h>
 
using namespace muduo;
using namespace muduo::net;
 
const int Connector::kMaxRetryDelayMs;
 
Connector::Connector(EventLoop* loop, const InetAddress& serverAddr)
  : loop_(loop),
    serverAddr_(serverAddr),
    connect_(false),
    state_(kDisconnected),
    retryDelayMs_(kInitRetryDelayMs)
{
  LOG_DEBUG << "ctor[" << this << "]";
}
 
Connector::~Connector()
{
  LOG_DEBUG << "dtor[" << this << "]";
  assert(!channel_);
}
 
// 可以跨线程调用 ，把重新发起的连接函数放到IO线程去，所以是线程
// 安全的
void Connector::start()
{
  connect_ = true;
  loop_->runInLoop(boost::bind(&Connector::startInLoop, this)); // FIXME: unsafe
}
 
/*重新发起连接*/
void Connector::startInLoop()
{
  loop_->assertInLoopThread();
  assert(state_ == kDisconnected);
  if (connect_)
  {
    /*如果为true 重新连接，
    如果调用stop 后，那么connect_ = false
    */
    connect();
  }
  else
  {
    LOG_DEBUG << "do not connect";
  }
}
 
/*让IO线程线程停止连接，这个连接处于*/
void Connector::stop()
{
  connect_ = false;
  loop_->runInLoop(boost::bind(&Connector::stopInLoop, this)); // FIXME: unsafe
  // FIXME: cancel timer
}
 
void Connector::stopInLoop()
{
  loop_->assertInLoopThread();
  if (state_ == kConnecting)
  {
    setState(kDisconnected);
    int sockfd = removeAndResetChannel();   // 将通道从poller中移除关注，并将channel置空
    retry(sockfd);      // 这里并非要重连，只是调用sockets::close(sockfd);
  }
}
 
/*发起连接  */
void Connector::connect()
{
  int sockfd = sockets::createNonblockingOrDie();   // 创建非阻塞套接字
  /*进行连接请求*/
  int ret = sockets::connect(sockfd, serverAddr_.getSockAddrInet());
 
  int savedErrno = (ret == 0) ? 0 : errno;
  switch (savedErrno)
  {
    case 0:
    case EINPROGRESS:   // 非阻塞套接字，未连接成功返回码是EINPROGRESS表示正在连接
    case EINTR:
    case EISCONN:           // 连接成功
      connecting(sockfd);
      break;
 
    case EAGAIN:
    case EADDRINUSE:
    case EADDRNOTAVAIL:
    case ECONNREFUSED:
    case ENETUNREACH:
      retry(sockfd);        // 重连
      break;
 
    case EACCES:
    case EPERM:
    case EAFNOSUPPORT:
    case EALREADY:
    case EBADF:
    case EFAULT:
    case ENOTSOCK:
      LOG_SYSERR << "connect error in Connector::startInLoop " << savedErrno;
      sockets::close(sockfd);   // 不能重连，关闭sockfd
      break;
 
    default:
      LOG_SYSERR << "Unexpected error in Connector::startInLoop " << savedErrno;
      sockets::close(sockfd);
      // connectErrorCallback_();
      break;
  }
}
 
// 不能跨线程调用
void Connector::restart()
{
  loop_->assertInLoopThread();
  setState(kDisconnected);
  retryDelayMs_ = kInitRetryDelayMs;
  connect_ = true;
  startInLoop();
}
 
void Connector::connecting(int sockfd)
{
  //不管是真正连接还是连接成功了，都把状态设置为kConnecting
  setState(kConnecting);
  assert(!channel_);
  // Channel与sockfd关联
  channel_.reset(new Channel(loop_, sockfd));
  // 设置可写回调函数，主要是为了“正在连接”状态设置的
  channel_->setWriteCallback(
      boost::bind(&Connector::handleWrite, this)); // FIXME: unsafe
  // 设置错误回调函数
  channel_->setErrorCallback(
      boost::bind(&Connector::handleError, this)); // FIXME: unsafe
 
  // channel_->tie(shared_from_this()); is not working,
  // as channel_ is not managed by shared_ptr
 
  channel_->enableWriting();     // 让Poller关注可写事件，writing一定产生，不管是连接成功还是连接失败，
                                //
}
 
int Connector::removeAndResetChannel()
{
  channel_->disableAll();
  channel_->remove();            // 从poller移除关注
  int sockfd = channel_->fd();
  // Can't reset channel_ here, because we are inside Channel::handleEvent
  // 不能在这里重置channel_，因为正在调用Channel::handleEvent
  loop_->queueInLoop(boost::bind(&Connector::resetChannel, this)); // FIXME: unsafe
  return sockfd;
}
 
void Connector::resetChannel()
{
  channel_.reset();     // channel_ 置空
}
 
 
void Connector::handleWrite()
{
  LOG_TRACE << "Connector::handleWrite " << state_;
  /*由于在Connector::connecting注册可写事件时，“状态” 被设置为kConnecting（不管完不完成连接），
  所以第一次handleWrite时，得把“状态”设置为kConnected*/
  if (state_ == kConnecting)
  {
    /*这里不用再关注可写事件了，这里也是防止busy loop*/
    int sockfd = removeAndResetChannel();   // 从poller中移除关注，并将channel置空
    // socket可写并不意味着连接一定建立成功
    // 还需要用getsockopt(sockfd, SOL_SOCKET, SO_ERROR, ...)再次确认一下。
    int err = sockets::getSocketError(sockfd);
    if (err)        // 有错误
    {
      LOG_WARN << "Connector::handleWrite - SO_ERROR = "
               << err << " " << strerror_tl(err);
      retry(sockfd);        // 重连
    }
    else if (sockets::isSelfConnect(sockfd))        // 自连接
    {
      LOG_WARN << "Connector::handleWrite - Self connect";
      retry(sockfd);        // 重连
    }
    else    // 连接成功
    {
      setState(kConnected);
      if (connect_)
      {
        newConnectionCallback_(sockfd);     // 回调连接成功的回调函数
      }
      else
      {
        sockets::close(sockfd);
      }
    }
  }
  else
  {
    // what happened?
    assert(state_ == kDisconnected);
  }
}
 
/*连接出错的处理函数*/
void Connector::handleError()
{
  LOG_ERROR << "Connector::handleError";
  assert(state_ == kConnecting);
 
  int sockfd = removeAndResetChannel();     // 从poller中移除关注，并将channel置空
  int err = sockets::getSocketError(sockfd);
  LOG_TRACE << "SO_ERROR = " << err << " " << strerror_tl(err);
  retry(sockfd);
}
 
// 采用back-off策略重连，即重连时间逐渐延长，0.5s, 1s, 2s, ...直至30s
void Connector::retry(int sockfd)
{
  sockets::close(sockfd);
  setState(kDisconnected);
  if (connect_)
  {
    LOG_INFO << "Connector::retry - Retry connecting to " << serverAddr_.toIpPort()
             << " in " << retryDelayMs_ << " milliseconds. ";
    // 注册一个定时操作，重连
    loop_->runAfter(retryDelayMs_/1000.0,
                    boost::bind(&Connector::startInLoop, shared_from_this()));
    retryDelayMs_ = std::min(retryDelayMs_ * 2, kMaxRetryDelayMs);
  }
  else
  {
    LOG_DEBUG << "do not connect";
  }
}

```
### 测试程序

``` c++
#include <muduo/net/Channel.h>
#include <muduo/net/TcpClient.h>
 
#include <muduo/base/Logging.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
 
#include <boost/bind.hpp>
 
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
class TestClient
{
 public:
  TestClient(EventLoop* loop, const InetAddress& listenAddr)
    : loop_(loop),
      client_(loop, listenAddr, "TestClient"),
      stdinChannel_(loop, 0)
  {
    client_.setConnectionCallback(
        boost::bind(&TestClient::onConnection, this, _1));
    client_.setMessageCallback(
        boost::bind(&TestClient::onMessage, this, _1, _2, _3));
    //client_.enableRetry();
    // 标准输入缓冲区中有数据的时候，回调TestClient::handleRead
    stdinChannel_.setReadCallback(boost::bind(&TestClient::handleRead, this));
    stdinChannel_.enableReading();
  }
 
  void connect()
  {
    client_.connect();
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
    else
    {
      printf("onConnection(): connection [%s] is down\n",
             conn->name().c_str());
    }
  }
 
  void onMessage(const TcpConnectionPtr& conn, Buffer* buf, Timestamp time)
  {
    string msg(buf->retrieveAllAsString());
    printf("onMessage(): recv a message [%s]\n", msg.c_str());
    LOG_TRACE << conn->name() << " recv " << msg.size() << " bytes at " << time.toFormattedString();
  }
 
  // 标准输入缓冲区中有数据的时候，回调该函数
  void handleRead()
  {
    char buf[1024] = {0};
    fgets(buf, 1024, stdin);
    buf[strlen(buf)-1] = '\0';      // 去除\n
    client_.connection()->send(buf);
  }
 
  EventLoop* loop_;
  TcpClient client_;
  Channel stdinChannel_;        // 标准输入Channel
};
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid() << ", tid = " << CurrentThread::tid();
  EventLoop loop;
  InetAddress serverAddr("127.0.0.1", 8888);
  TestClient client(&loop, serverAddr);
  client.connect();
  loop.loop();
}
```

## [40] http 简单描述 

### http request

`request line` + `header` + `body` (`header`分为普通报头，请求报头与实体报头) `header`与`body`之间有一空行(`CRLF`)

#### 请求方法有 

`Get`, `Post`, `Head`, `Put`, `Delete`等

协议版本号：`1.0`, `1.1`

#### 常用请求头

- Accept：浏览器可接受的媒体（MIME）类型；
- Accept-Language：浏览器所希望的语言种类
- Accept-Encoding：浏览器能够解码的编码方法，如gzip，deflate等
- User-Agent：告诉HTTP服务器， 客户端使用的操作系统和浏览器的名称和版本
- Connection：表示是否需要持久连接，Keep-Alive表示长连接，close表示短连接

### http response

status line + header + body （header分为普通报头，响应报头与实体报头）
header与body之间有一空行（CRLF）

状态响应码

- 1XX  提示信息 
表示请求已被成功接收，继续处理
- 2XX  成功 
表示请求已被成功接收，理解，接受
- 3XX  重定向 
要完成请求必须进行更进一步的处理
- 4XX  客户端错误 
请求有语法错误或请求无法实现
- 5XX  服务器端错误 
服务器执行一个有效请求失败

### 一个典型的http请求

``` c++
GET / HTTP/1.1 (请求行)
Accept: image/jpeg, application/x-ms-application, image/gif, application/xaml+xml, image/pjpeg, application/x-ms-xbap, application/vnd.ms-excel, application/vnd.ms-powerpoint, application/msword, */*(请求头)
Accept-Language: zh-CN(请求头)
User-Agent: Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.1; Trident/4.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; Tablet PC 2.0)(请求头)
Accept-Encoding: gzip, deflate(请求头)
Host: 192.168.159.188:800(请求头)
Connection: Keep-Alive(请求头)

/*当前是get请求，没有body，如果是post请求的话会有实体*/
```

### 一个典型的http应答

``` c++
HTTP/1.1 200 OK(请求行)
Content-Length: 112(头部)
Connection: Keep-Alive(头部)
Content-Type: text/html(头部)
Server: Muduo(头部)

<html><head><title>This is title</title></head><body><h1>Hello</h1>Now is 20130611 02:14:31.518462</body></html>(实体)
```

### muduo_http库涉及到的类

```c++
HttpRequest：http请求类封装
HttpResponse：http响应类封装
HttpContext：http协议解析类
HttpServer：http服务器类封装
```

## [40-1] http实现源码

### HttpRequest头文件(包含实现)

HttpResquest.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_HTTP_HTTPREQUEST_H
#define MUDUO_NET_HTTP_HTTPREQUEST_H
 
#include <muduo/base/copyable.h>
#include <muduo/base/Timestamp.h>
#include <muduo/base/Types.h>
 
#include <map>
#include <assert.h>
#include <stdio.h>
 
namespace muduo
{
namespace net
{
 
class HttpRequest : public muduo::copyable
{
 public:
  enum Method
  {
    kInvalid, kGet, kPost, kHead, kPut, kDelete
  };
  enum Version
  {
    kUnknown, kHttp10, kHttp11
  };
 
  HttpRequest()
    : method_(kInvalid),
      version_(kUnknown)
  {
  }
 
  void setVersion(Version v)
  {
    version_ = v;
  }
 
  Version getVersion() const
  { return version_; }
 
/*设置请求方法*/
  bool setMethod(const char* start, const char* end)
  {
    assert(method_ == kInvalid);
    string m(start, end);
    if (m == "GET")
    {
      method_ = kGet;
    }
    else if (m == "POST")
    {
      method_ = kPost;
    }
    else if (m == "HEAD")
    {
      method_ = kHead;
    }
    else if (m == "PUT")
    {
      method_ = kPut;
    }
    else if (m == "DELETE")
    {
      method_ = kDelete;
    }
    else
    {
      method_ = kInvalid;
    }
    return method_ != kInvalid;
  }
 
/*返回请求方法*/
  Method method() const
  { return method_; }
 
/*请求方法转为字符串*/
  const char* methodString() const
  {
    const char* result = "UNKNOWN";
    switch(method_)
    {
      case kGet:
        result = "GET";
        break;
      case kPost:
        result = "POST";
        break;
      case kHead:
        result = "HEAD";
        break;
      case kPut:
        result = "PUT";
        break;
      case kDelete:
        result = "DELETE";
        break;
      default:
        break;
    }
    return result;
  }
 
/*设置请求路径*/
  void setPath(const char* start, const char* end)
  {
    path_.assign(start, end);
  }
 
/*返回请求路径*/
  const string& path() const
  { return path_; }
 
/*设置接收时间*/
  void setReceiveTime(Timestamp t)
  { receiveTime_ = t; }
 
  Timestamp receiveTime() const
  { return receiveTime_; }
 
/*添加一个头部信息*/
  /*
Accept-Language(start):(colon) zh-CN(end)
 
-->Accept-Language
   zh-CN
*/
  void addHeader(const char* start, const char* colon, const char* end)
  {
    string field(start, colon);     // header域
    ++colon;
    // 去除左空格
    while (colon < end && isspace(*colon))
    {
      ++colon;
    }
    string value(colon, end);       // header值
    // 去除右空格
    while (!value.empty() && isspace(value[value.size()-1]))
    {
      value.resize(value.size()-1);
    }
    headers_[field] = value;
  }
 
/*根据头域，获得头域的信息*/
  string getHeader(const string& field) const
  {
    string result;
    std::map<string, string>::const_iterator it = headers_.find(field);
    if (it != headers_.end())
    {
      result = it->second;
    }
    return result;
  }
 
  const std::map<string, string>& headers() const
  { return headers_; }
 
  void swap(HttpRequest& that)
  {
    /*貌似少了version的交换*/
    std::swap(method_, that.method_);
    path_.swap(that.path_);
    receiveTime_.swap(that.receiveTime_);
    headers_.swap(that.headers_);
  }
 
 private:
  Method method_;       // 请求方法
  Version version_;     // 协议版本1.0/1.1
  string path_;         // 请求路径
  Timestamp receiveTime_;   // 请求时间
  std::map<string, string> headers_;  // header列表
};
 
}
}
 
#endif  // MUDUO_NET_HTTP_HTTPREQUEST_H
```

### HttpResponse头文件

HttpResponse.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_HTTP_HTTPRESPONSE_H
#define MUDUO_NET_HTTP_HTTPRESPONSE_H
 
#include <muduo/base/copyable.h>
#include <muduo/base/Types.h>
 
#include <map>
 
namespace muduo
{
namespace net
{
 
class Buffer;
class HttpResponse : public muduo::copyable
{
 public:
  enum HttpStatusCode
  {
    kUnknown,
    k200Ok = 200,       // 成功
    k301MovedPermanently = 301,     // 301重定向，请求的页面永久性移至另一个地址
    k400BadRequest = 400,           // 错误的请求，语法格式有错，服务器无法处理此请求
    k404NotFound = 404,     // 请求的网页不存在
  };
 
  explicit HttpResponse(bool close)
    : statusCode_(kUnknown),
      closeConnection_(close)
  {
  }
 
  void setStatusCode(HttpStatusCode code)
  { statusCode_ = code; }
 
  void setStatusMessage(const string& message)
  { statusMessage_ = message; }
 
  void setCloseConnection(bool on)
  { closeConnection_ = on; }
 
  bool closeConnection() const
  { return closeConnection_; }
 
  // 设置文档媒体类型（MIME）
  void setContentType(const string& contentType)
  { addHeader("Content-Type", contentType); }
 
  // FIXME: replace string with StringPiece
  void addHeader(const string& key, const string& value)
  { headers_[key] = value; }
 
  void setBody(const string& body)
  { body_ = body; }
 
  void appendToBuffer(Buffer* output) const;    // 将HttpResponse添加到Buffer
 
 private:
  std::map<string, string> headers_;  // header列表
  HttpStatusCode statusCode_;           // 状态响应码
  // FIXME: add http version
  string statusMessage_;                // 状态响应码对应的文本信息
  bool closeConnection_;                // 是否关闭连接
  string body_;                         // 实体
};
 
}
}
 
#endif  // MUDUO_NET_HTTP_HTTPRESPONSE_H
```

### HttpResponse源文件

HttpResponse.cc

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
 
#include <muduo/net/http/HttpResponse.h>
#include <muduo/net/Buffer.h>
 
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
void HttpResponse::appendToBuffer(Buffer* output) const
{
  char buf[32];
  // 添加响应头
  snprintf(buf, sizeof buf, "HTTP/1.1 %d ", statusCode_);
  output->append(buf);
  output->append(statusMessage_);
  output->append("\r\n");
 
  if (closeConnection_)
  {
    // 如果是短连接，不需要告诉浏览器Content-Length，浏览器也能正确处理
    // 短链接不会出现粘包问题
    output->append("Connection: close\r\n");
  }
  else
  {
    snprintf(buf, sizeof buf, "Content-Length: %zd\r\n", body_.size()); // 实体长度
    output->append(buf);
    output->append("Connection: Keep-Alive\r\n");
  }
 
  // header列表
  for (std::map<string, string>::const_iterator it = headers_.begin();
       it != headers_.end();
       ++it)
  {
    output->append(it->first);
    output->append(": ");
    output->append(it->second);
    output->append("\r\n");
  }
 
  output->append("\r\n");    // header与body之间的空行
  output->append(body_);
}
```
### HttpContext文件

HttpContext.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is an internal header file, you should not include this.
 
#ifndef MUDUO_NET_HTTP_HTTPCONTEXT_H
#define MUDUO_NET_HTTP_HTTPCONTEXT_H
 
#include <muduo/base/copyable.h>
 
#include <muduo/net/http/HttpRequest.h>
 
namespace muduo
{
namespace net
{
 
class HttpContext : public muduo::copyable
{
 public:
  enum HttpRequestParseState
  {
    kExpectRequestLine,//正处于请求行的状态
    kExpectHeaders,   //正处于请求头的状态
    kExpectBody,      //正处于请求实体的状态
    kGotAll,          //全部都解析完毕
  };
 
  HttpContext()
    : state_(kExpectRequestLine)
  {
  }
 
  // default copy-ctor, dtor and assignment are fine
 
  bool expectRequestLine() const
  { return state_ == kExpectRequestLine; }
 
  bool expectHeaders() const
  { return state_ == kExpectHeaders; }
 
  bool expectBody() const
  { return state_ == kExpectBody; }
 
  bool gotAll() const
  { return state_ == kGotAll; }
 
  void receiveRequestLine()
  { state_ = kExpectHeaders; }
  /*注意这里没有处理body实体的请求*/
  void receiveHeaders()
  { state_ = kGotAll; }  // FIXME
 
  // 重置HttpContext状态
  void reset()
  {
    state_ = kExpectRequestLine;
    HttpRequest dummy;
    request_.swap(dummy);
  }
 
  const HttpRequest& request() const
  { return request_; }
 
  HttpRequest& request()
  { return request_; }
 
 private:
  HttpRequestParseState state_;     // 请求解析状态
  HttpRequest request_;             // http请求
};
 
}
}
 
#endif  // MUDUO_NET_HTTP_HTTPCONTEXT_H
```

### HttpServer头文件

HttpServer.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_HTTP_HTTPSERVER_H
#define MUDUO_NET_HTTP_HTTPSERVER_H
 
#include <muduo/net/TcpServer.h>
#include <boost/noncopyable.hpp>
 
namespace muduo
{
namespace net
{
 
class HttpRequest;
class HttpResponse;
 
/// A simple embeddable HTTP server designed for report status of a program.
/// It is not a fully HTTP 1.1 compliant server, but provides minimum features
/// that can communicate with HttpClient and Web browser.
/// It is synchronous, just like Java Servlet.
class HttpServer : boost::noncopyable
{
 public:
  typedef boost::function<void (const HttpRequest&,
                                HttpResponse*)> HttpCallback;
 
  HttpServer(EventLoop* loop,
             const InetAddress& listenAddr,
             const string& name);
 
  ~HttpServer();  // force out-line dtor, for scoped_ptr members.
 
  /// Not thread safe, callback be registered before calling start().
  void setHttpCallback(const HttpCallback& cb)
  {
    httpCallback_ = cb;
  }
 
  /*subIO thread numbers*/
  void setThreadNum(int numThreads)
  {
    server_.setThreadNum(numThreads);
  }
 
  void start();
 
 private:
  void onConnection(const TcpConnectionPtr& conn);
  /*当以http请求到来时
onMessage--->onRequest--->HttpCallback
  */
  void onMessage(const TcpConnectionPtr& conn,
                 Buffer* buf,
                 Timestamp receiveTime);
  void onRequest(const TcpConnectionPtr&, const HttpRequest&);
 
  TcpServer server_;
  HttpCallback httpCallback_;   // 在处理http请求（即调用onRequest）的过程中回调此函数，对请求进行具体的处理
};
 
}
}
 
#endif  // MUDUO_NET_HTTP_HTTPSERVER_H
````

### HttpServer源文件

HttpServer.cc

``` c++

// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
 
#include <muduo/net/http/HttpServer.h>
 
#include <muduo/base/Logging.h>
#include <muduo/net/http/HttpContext.h>
#include <muduo/net/http/HttpRequest.h>
#include <muduo/net/http/HttpResponse.h>
 
#include <boost/bind.hpp>
 
using namespace muduo;
using namespace muduo::net;
/*
GET / HTTP/1.1 (请求行)
Accept: image/jpeg, application/x-ms-application, image/gif, application/xaml+xml, image/pjpeg, application/x-ms-xbap, application/vnd.ms-excel, application/vnd.ms-powerpoint, application/msword(请求头)
Accept-Language: zh-CN(请求头)
User-Agent: Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.1; Trident/4.0; SLCC2; .NET CLR 2.0.50727; .NET CLR 3.5.30729; .NET CLR 3.0.30729; Media Center PC 6.0; Tablet PC 2.0)(请求头)
Accept-Encoding: gzip, deflate(请求头)
Host: 192.168.159.188:800(请求头)
Connection: Keep-Alive(请求头)
 
*/
namespace muduo
{
namespace net
{
namespace detail
{
 
// FIXME: move to HttpContext class
bool processRequestLine(const char* begin, const char* end, HttpContext* context)
{
  bool succeed = false;
  const char* start = begin;
  const char* space = std::find(start, end, ' ');
  HttpRequest& request = context->request();
  if (space != end && request.setMethod(start, space))      // 解析请求方法
  {
    start = space+1;
    space = std::find(start, end, ' ');
    if (space != end)
    {
      request.setPath(start, space);    // 解析PATH
      start = space+1;
      succeed = end-start == 8 && std::equal(start, end-1, "HTTP/1.");
      if (succeed)
      {
        if (*(end-1) == '1')
        {
          request.setVersion(HttpRequest::kHttp11);     // HTTP/1.1
        }
        else if (*(end-1) == '0')
        {
          request.setVersion(HttpRequest::kHttp10);     // HTTP/1.0
        }
        else
        {
          succeed = false;
        }
      }
    }
  }
  return succeed;
}
 
// FIXME: move to HttpContext class
// return false if any error
// 开始解析请求
bool parseRequest(Buffer* buf, HttpContext* context, Timestamp receiveTime)
{
  bool ok = true;
  bool hasMore = true;
  while (hasMore)
  {
    if (context->expectRequestLine())    // 处于解析请求行状态
    {
      const char* crlf = buf->findCRLF();
      if (crlf)
      {
        ok = processRequestLine(buf->peek(), crlf, context); // 解析请求行 ，*crlf ='\r'
        if (ok)
        {
          context->request().setReceiveTime(receiveTime);        // 设置请求时间
          buf->retrieveUntil(crlf + 2);      // 将请求行从buf中取回，包括\r\n
          context->receiveRequestLine(); // 将HttpContext状态改为kExpectHeaders
        }
        else
        {
          hasMore = false;
        }
      }
      else
      {
        hasMore = false;
      }
    }
    else if (context->expectHeaders())       // 解析header
    {
      const char* crlf = buf->findCRLF();
      if (crlf)
      {
        const char* colon = std::find(buf->peek(), crlf, ':');       //冒号所在位置
        if (colon != crlf)
        {
          context->request().addHeader(buf->peek(), colon, crlf);
        }
        else
        {
          // empty line, end of header
          context->receiveHeaders();     // HttpContext将状态改为kGotAll
          hasMore = !context->gotAll();
        }
        buf->retrieveUntil(crlf + 2);        // 将header从buf中取回，包括\r\n
      }
      else
      {
        hasMore = false;
      }
    }
    else if (context->expectBody())          // 当前还不支持请求中带body
    {
      // FIXME:
    }
  }
  return ok;
}
 
void defaultHttpCallback(const HttpRequest&, HttpResponse* resp)
{
  resp->setStatusCode(HttpResponse::k404NotFound);
  resp->setStatusMessage("Not Found");
  resp->setCloseConnection(true);
}
 
}
}
}
 
HttpServer::HttpServer(EventLoop* loop,
                       const InetAddress& listenAddr,
                       const string& name)
  : server_(loop, listenAddr, name),
    httpCallback_(detail::defaultHttpCallback)
{
  server_.setConnectionCallback(
      boost::bind(&HttpServer::onConnection, this, _1));
  server_.setMessageCallback(
      boost::bind(&HttpServer::onMessage, this, _1, _2, _3));
}
 
HttpServer::~HttpServer()
{
}
 
void HttpServer::start()
{
  LOG_WARN << "HttpServer[" << server_.name()
    << "] starts listenning on " << server_.hostport();
  server_.start();
}
 
void HttpServer::onConnection(const TcpConnectionPtr& conn)
{
  if (conn->connected())
  {
    //boost any
    conn->setContext(HttpContext()); // TcpConnection与一个HttpContext绑定
  }
}
 
void HttpServer::onMessage(const TcpConnectionPtr& conn,
                           Buffer* buf,
                           Timestamp receiveTime)
{
  HttpContext* context = boost::any_cast<HttpContext>(conn->getMutableContext());
 
  if (!detail::parseRequest(buf, context, receiveTime))
  {
    conn->send("HTTP/1.1 400 Bad Request\r\n\r\n");
    conn->shutdown();
  }
 
  // 请求消息解析完毕
  if (context->gotAll())
  {
    onRequest(conn, context->request());
    context->reset();        // 本次请求处理完毕，重置HttpContext，适用于长连接
  }
}
 
void HttpServer::onRequest(const TcpConnectionPtr& conn, const HttpRequest& req)
{
  const string& connection = req.getHeader("Connection");
  //是否关闭连接
  bool close = connection == "close" ||
    (req.getVersion() == HttpRequest::kHttp10 && connection != "Keep-Alive");
  HttpResponse response(close);
  httpCallback_(req, &response);//回调用户函数
  Buffer buf;
  response.appendToBuffer(&buf);
  conn->send(&buf);
  if (response.closeConnection())
  {
    conn->shutdown();
  }
}
```

### 测试程序

``` c++
#include <muduo/net/http/HttpServer.h>
#include <muduo/net/http/HttpRequest.h>
#include <muduo/net/http/HttpResponse.h>
#include <muduo/net/EventLoop.h>
#include <muduo/base/Logging.h>
 
#include <iostream>
#include <map>
 
using namespace muduo;
using namespace muduo::net;
 
extern char favicon[555];
bool benchmark = false;
 
// 实际的请求处理
void onRequest(const HttpRequest& req, HttpResponse* resp)
{
  std::cout << "Headers " << req.methodString() << " " << req.path() << std::endl;
  if (!benchmark)
  {
    const std::map<string, string>& headers = req.headers();
    for (std::map<string, string>::const_iterator it = headers.begin();
         it != headers.end();
         ++it)
    {
      std::cout << it->first << ": " << it->second << std::endl;
    }
  }
 
  if (req.path() == "/")
  {
    resp->setStatusCode(HttpResponse::k200Ok);
    resp->setStatusMessage("OK");
    resp->setContentType("text/html");
    resp->addHeader("Server", "Muduo");
    string now = Timestamp::now().toFormattedString();
    resp->setBody("<html><head><title>This is title</title></head>"
        "<body><h1>Hello</h1>Now is " + now +
        "</body></html>");
  }
  else if (req.path() == "/favicon.ico")
  {
    resp->setStatusCode(HttpResponse::k200Ok);
    resp->setStatusMessage("OK");
    resp->setContentType("image/png");
    resp->setBody(string(favicon, sizeof favicon));
  }
  else if (req.path() == "/hello")
  {
    resp->setStatusCode(HttpResponse::k200Ok);
    resp->setStatusMessage("OK");
    resp->setContentType("text/plain");
    resp->addHeader("Server", "Muduo");
    resp->setBody("hello, world!\n");
  }
  else
  {
    resp->setStatusCode(HttpResponse::k404NotFound);
    resp->setStatusMessage("Not Found");
    resp->setCloseConnection(true);
  }
}
 
int main(int argc, char* argv[])
{
  int numThreads = 0;
  if (argc > 1)
  {
    benchmark = true;
    Logger::setLogLevel(Logger::WARN);
    numThreads = atoi(argv[1]);
  }
  EventLoop loop;
  HttpServer server(&loop, InetAddress(8000), "dummy");
  server.setHttpCallback(onRequest);
  server.setThreadNum(numThreads);
  server.start();
  loop.loop();
}
 
// 这是一个图片数据
char favicon[555] = {
  '\x89', 'P', 'N', 'G', '\xD', '\xA', '\x1A', '\xA',
  '\x0', '\x0', '\x0', '\xD', 'I', 'H', 'D', 'R',
  '\x0', '\x0', '\x0', '\x10', '\x0', '\x0', '\x0', '\x10',
  '\x8', '\x6', '\x0', '\x0', '\x0', '\x1F', '\xF3', '\xFF',
  'a', '\x0', '\x0', '\x0', '\x19', 't', 'E', 'X',
  't', 'S', 'o', 'f', 't', 'w', 'a', 'r',
  'e', '\x0', 'A', 'd', 'o', 'b', 'e', '\x20',
  'I', 'm', 'a', 'g', 'e', 'R', 'e', 'a',
  'd', 'y', 'q', '\xC9', 'e', '\x3C', '\x0', '\x0',
  '\x1', '\xCD', 'I', 'D', 'A', 'T', 'x', '\xDA',
  '\x94', '\x93', '9', 'H', '\x3', 'A', '\x14', '\x86',
  '\xFF', '\x5D', 'b', '\xA7', '\x4', 'R', '\xC4', 'm',
  '\x22', '\x1E', '\xA0', 'F', '\x24', '\x8', '\x16', '\x16',
  'v', '\xA', '6', '\xBA', 'J', '\x9A', '\x80', '\x8',
  'A', '\xB4', 'q', '\x85', 'X', '\x89', 'G', '\xB0',
  'I', '\xA9', 'Q', '\x24', '\xCD', '\xA6', '\x8', '\xA4',
  'H', 'c', '\x91', 'B', '\xB', '\xAF', 'V', '\xC1',
  'F', '\xB4', '\x15', '\xCF', '\x22', 'X', '\x98', '\xB',
  'T', 'H', '\x8A', 'd', '\x93', '\x8D', '\xFB', 'F',
  'g', '\xC9', '\x1A', '\x14', '\x7D', '\xF0', 'f', 'v',
  'f', '\xDF', '\x7C', '\xEF', '\xE7', 'g', 'F', '\xA8',
  '\xD5', 'j', 'H', '\x24', '\x12', '\x2A', '\x0', '\x5',
  '\xBF', 'G', '\xD4', '\xEF', '\xF7', '\x2F', '6', '\xEC',
  '\x12', '\x20', '\x1E', '\x8F', '\xD7', '\xAA', '\xD5', '\xEA',
  '\xAF', 'I', '5', 'F', '\xAA', 'T', '\x5F', '\x9F',
  '\x22', 'A', '\x2A', '\x95', '\xA', '\x83', '\xE5', 'r',
  '9', 'd', '\xB3', 'Y', '\x96', '\x99', 'L', '\x6',
  '\xE9', 't', '\x9A', '\x25', '\x85', '\x2C', '\xCB', 'T',
  '\xA7', '\xC4', 'b', '1', '\xB5', '\x5E', '\x0', '\x3',
  'h', '\x9A', '\xC6', '\x16', '\x82', '\x20', 'X', 'R',
  '\x14', 'E', '6', 'S', '\x94', '\xCB', 'e', 'x',
  '\xBD', '\x5E', '\xAA', 'U', 'T', '\x23', 'L', '\xC0',
  '\xE0', '\xE2', '\xC1', '\x8F', '\x0', '\x9E', '\xBC', '\x9',
  'A', '\x7C', '\x3E', '\x1F', '\x83', 'D', '\x22', '\x11',
  '\xD5', 'T', '\x40', '\x3F', '8', '\x80', 'w', '\xE5',
  '3', '\x7', '\xB8', '\x5C', '\x2E', 'H', '\x92', '\x4',
  '\x87', '\xC3', '\x81', '\x40', '\x20', '\x40', 'g', '\x98',
  '\xE9', '6', '\x1A', '\xA6', 'g', '\x15', '\x4', '\xE3',
  '\xD7', '\xC8', '\xBD', '\x15', '\xE1', 'i', '\xB7', 'C',
  '\xAB', '\xEA', 'x', '\x2F', 'j', 'X', '\x92', '\xBB',
  '\x18', '\x20', '\x9F', '\xCF', '3', '\xC3', '\xB8', '\xE9',
  'N', '\xA7', '\xD3', 'l', 'J', '\x0', 'i', '6',
  '\x7C', '\x8E', '\xE1', '\xFE', 'V', '\x84', '\xE7', '\x3C',
  '\x9F', 'r', '\x2B', '\x3A', 'B', '\x7B', '7', 'f',
  'w', '\xAE', '\x8E', '\xE', '\xF3', '\xBD', 'R', '\xA9',
  'd', '\x2', 'B', '\xAF', '\x85', '2', 'f', 'F',
  '\xBA', '\xC', '\xD9', '\x9F', '\x1D', '\x9A', 'l', '\x22',
  '\xE6', '\xC7', '\x3A', '\x2C', '\x80', '\xEF', '\xC1', '\x15',
  '\x90', '\x7', '\x93', '\xA2', '\x28', '\xA0', 'S', 'j',
  '\xB1', '\xB8', '\xDF', '\x29', '5', 'C', '\xE', '\x3F',
  'X', '\xFC', '\x98', '\xDA', 'y', 'j', 'P', '\x40',
  '\x0', '\x87', '\xAE', '\x1B', '\x17', 'B', '\xB4', '\x3A',
  '\x3F', '\xBE', 'y', '\xC7', '\xA', '\x26', '\xB6', '\xEE',
  '\xD9', '\x9A', '\x60', '\x14', '\x93', '\xDB', '\x8F', '\xD',
  '\xA', '\x2E', '\xE9', '\x23', '\x95', '\x29', 'X', '\x0',
  '\x27', '\xEB', 'n', 'V', 'p', '\xBC', '\xD6', '\xCB',
  '\xD6', 'G', '\xAB', '\x3D', 'l', '\x7D', '\xB8', '\xD2',
  '\xDD', '\xA0', '\x60', '\x83', '\xBA', '\xEF', '\x5F', '\xA4',
  '\xEA', '\xCC', '\x2', 'N', '\xAE', '\x5E', 'p', '\x1A',
  '\xEC', '\xB3', '\x40', '9', '\xAC', '\xFE', '\xF2', '\x91',
  '\x89', 'g', '\x91', '\x85', '\x21', '\xA8', '\x87', '\xB7',
  'X', '\x7E', '\x7E', '\x85', '\xBB', '\xCD', 'N', 'N',
  'b', 't', '\x40', '\xFA', '\x93', '\x89', '\xEC', '\x1E',
  '\xEC', '\x86', '\x2', 'H', '\x26', '\x93', '\xD0', 'u',
  '\x1D', '\x7F', '\x9', '2', '\x95', '\xBF', '\x1F', '\xDB',
  '\xD7', 'c', '\x8A', '\x1A', '\xF7', '\x5C', '\xC1', '\xFF',
  '\x22', 'J', '\xC3', '\x87', '\x0', '\x3', '\x0', 'K',
  '\xBB', '\xF8', '\xD6', '\x2A', 'v', '\x98', 'I', '\x0',
  '\x0', '\x0', '\x0', 'I', 'E', 'N', 'D', '\xAE',
  'B', '\x60', '\x82',
};
```

## [41] muduo_inspect库源码分析

- muduo_inspect库通过HTTP方式为服务器提供监控接口
- 接受了多少个TCP连接
- 当前有多少个活动连接
- 一共响应了多少次请求
- 每次请求的平均响应时间多少毫秒

- Inspector     // 包含了一个HttpServer对象
- ProcessInspector // 通过ProcessInfo返回进程信息
- ProcessInfo // 获取进程相关信息

### ProcessInspectort头文件

ProcessInspector.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is an internal header file, you should not include this.
 
#ifndef MUDUO_NET_INSPECT_PROCESSINSPECTOR_H
#define MUDUO_NET_INSPECT_PROCESSINSPECTOR_H
 
#include <muduo/net/inspect/Inspector.h>
#include <boost/noncopyable.hpp>
 
namespace muduo
{
namespace net
{
 
class ProcessInspector : boost::noncopyable
{
 public:
  void registerCommands(Inspector* ins);    // 注册命令接口
 
 private:
  static string pid(HttpRequest::Method, const Inspector::ArgList&);
  static string procStatus(HttpRequest::Method, const Inspector::ArgList&);
  static string openedFiles(HttpRequest::Method, const Inspector::ArgList&);
  static string threads(HttpRequest::Method, const Inspector::ArgList&);
 
};
 
}
}
 
#endif  // MUDUO_NET_INSPECT_PROCESSINSPECTOR_H
```

### ProcessInspectort源文件

ProcessInspector.cc

```
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
 
#include <muduo/net/inspect/ProcessInspector.h>
#include <muduo/base/ProcessInfo.h>
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
/*命令注册*/
void ProcessInspector::registerCommands(Inspector* ins)
{
  /*muduo             :   proc
    command           :   pid
    commandCallback   :   pid
    commnadHelp       :   print id
  */
  ins->add("proc", "pid", ProcessInspector::pid, "print pid");
  ins->add("proc", "status", ProcessInspector::procStatus, "print /proc/self/status");
  ins->add("proc", "opened_files", ProcessInspector::openedFiles, "count /proc/self/fd");
  ins->add("proc", "threads", ProcessInspector::threads, "list /proc/self/task");
}
 
string ProcessInspector::pid(HttpRequest::Method, const Inspector::ArgList&)
{
  char buf[32];
  snprintf(buf, sizeof buf, "%d", ProcessInfo::pid());
  return buf;
}
 
string ProcessInspector::procStatus(HttpRequest::Method, const Inspector::ArgList&)
{
  return ProcessInfo::procStatus();
}
 
string ProcessInspector::openedFiles(HttpRequest::Method, const Inspector::ArgList&)
{
  char buf[32];
  snprintf(buf, sizeof buf, "%d", ProcessInfo::openedFiles());
  return buf;
}
 
string ProcessInspector::threads(HttpRequest::Method, const Inspector::ArgList&)
{
  std::vector<pid_t> threads = ProcessInfo::threads();
  string result;
  for (size_t i = 0; i < threads.size(); ++i)
  {
    char buf[32];
    snprintf(buf, sizeof buf, "%d\n", threads[i]);
    result += buf;
  }
  return result;
}
```

### Inspector头文件

Inspector.h

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
// This is a public header file, it must only include public header files.
 
#ifndef MUDUO_NET_INSPECT_INSPECTOR_H
#define MUDUO_NET_INSPECT_INSPECTOR_H
 
#include <muduo/base/Mutex.h>
#include <muduo/net/http/HttpRequest.h>
#include <muduo/net/http/HttpServer.h>
 
#include <map>
#include <boost/function.hpp>
#include <boost/noncopyable.hpp>
#include <boost/scoped_ptr.hpp>
 
namespace muduo
{
namespace net
{
 
class ProcessInspector;
 
// A internal inspector of the running process, usually a singleton.
class Inspector : boost::noncopyable
{
 public:
  typedef std::vector<string> ArgList;
  typedef boost::function<string (HttpRequest::Method, const ArgList& args)> Callback;
  Inspector(EventLoop* loop,
            const InetAddress& httpAddr,
            const string& name);
  ~Inspector();
 
  // 如add("proc", "pid", ProcessInspector::pid, "print pid");
  // http://192.168.159.188:12345/proc/pid这个http请求就会相应的调用ProcessInspector::pid来处理
  void add(const string& module,
           const string& command,
           const Callback& cb,
           const string& help);
 
 private:
  typedef std::map<string, Callback> CommandList;
  typedef std::map<string, string> HelpList;
 
  void start();
  void onRequest(const HttpRequest& req, HttpResponse* resp);
 
  HttpServer server_;
  boost::scoped_ptr<ProcessInspector> processInspector_;
  MutexLock mutex_;
  std::map<string, CommandList> commands_;
  std::map<string, HelpList> helps_;
  /*
commands_ --- > <muduo commandlist>
helps_    ----> <muduo helplist>
 
commandlist---> <command callback>
HelpList  ----> <command helplist>
 
commands_[][]=xx
helps_[][]=xxx
 
  */
};
 
}
}
 
#endif  // MUDUO_NET_INSPECT_INSPECTOR_H
```

### Inspector源文件

Inspector.cc

``` c++
// Copyright 2010, Shuo Chen.  All rights reserved.
// http://code.google.com/p/muduo/
//
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
 
// Author: Shuo Chen (chenshuo at chenshuo dot com)
//
 
#include <muduo/net/inspect/Inspector.h>
 
#include <muduo/base/Thread.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/http/HttpRequest.h>
#include <muduo/net/http/HttpResponse.h>
#include <muduo/net/inspect/ProcessInspector.h>
 
//#include <iostream>
//#include <iterator>
//#include <sstream>
#include <boost/bind.hpp>
#include <boost/algorithm/string/classification.hpp>
#include <boost/algorithm/string/split.hpp>
 
using namespace muduo;
using namespace muduo::net;
 
namespace
{
Inspector* g_globalInspector = 0;
 
// Looks buggy
std::vector<string> split(const string& str)
{
  std::vector<string> result;
  size_t start = 0;
  size_t pos = str.find('/');
  while (pos != string::npos)
  {
    if (pos > start)
    {
      result.push_back(str.substr(start, pos-start));
    }
    start = pos+1;
    pos = str.find('/', start);
  }
 
  if (start < str.length())      // 说明最后一个字符不是'/'
  {
    result.push_back(str.substr(start));
  }
 
  return result;
}
 
}
 
Inspector::Inspector(EventLoop* loop,
                     const InetAddress& httpAddr,
                     const string& name)
    : server_(loop, httpAddr, "Inspector:"+name),
      processInspector_(new ProcessInspector)
{
  //断言在主线程当中
  assert(CurrentThread::isMainThread());
  assert(g_globalInspector == 0);
  g_globalInspector = this;
  /*设置请求的回调函数，这里的请求是完成了协议的解析之后*/
  server_.setHttpCallback(boost::bind(&Inspector::onRequest, this, _1, _2));
  /*注册命令*/
  processInspector_->registerCommands(this);
  // 这样子做法是为了防止竞态问题
  // 如果直接调用start，（当前线程不是loop所属的IO线程，是主线程）那么有可能，当前构造函数还没返回，
  // HttpServer所在的IO线程可能已经收到了http客户端的请求了（因为这时候HttpServer已启动），那么就会回调
  // Inspector::onRequest，而这时候构造函数还没返回，也就是说对象还没完全构造好。那么就会出现问题了
  loop->runAfter(0, boost::bind(&Inspector::start, this)); // little race condition
}
 
Inspector::~Inspector()
{
  assert(CurrentThread::isMainThread());
  g_globalInspector = NULL;
}
 
void Inspector::add(const string& module,
                    const string& command,
                    const Callback& cb,
                    const string& help)
{
  /*这里要不要加锁？？
因为在构造函数时 程序是注册完 processInspector_->registerCommands(this); 才执行
loop->runAfter(0, boost::bind(&Inspector::start, this)); // little race condition。
 
如果程序进行扩充时，就要了。
class TcpInspector : boost::noncopyable
{
 public:
  void registerCommands(Inspector* ins);  // 注册命令接口
 
 private:
  static string pid(HttpRequest::Method, const Inspector::ArgList&);
  static string procStatus(HttpRequest::Method, const Inspector::ArgList&);
  static string openedFiles(HttpRequest::Method, const Inspector::ArgList&);
  static string threads(HttpRequest::Method, const Inspector::ArgList&);
 
};
 
MyInspector ：public ProcessInspector
{
   TcpInspector
} ;
那么
MyInspector{
 
  HttpServer server_;
  boost::scoped_ptr<ProcessInspector> processInspector;
  boost::scoped_ptr<ProcessInspector> TcpInspector;
  MutexLock mutex_;
  std::map<string, CommandList> commands_;
  std::map<string, HelpList> helps_;
}
那么MyInspector 在初始化时，会初始化ProcessInspector 和TcpInspector，
如果ProcessInspector初始化完毕后，启动了OnreRequest(),而TcpInspector还没注册完，
也就是说TcpInspector还在add（）函数中，那么不加锁的话，就会出现问题了。
  */
  MutexLockGuard lock(mutex_);
  commands_[module][command] = cb;
  helps_[module][command] = help;
}
 
void Inspector::start()
{
  server_.start();
}
 
void Inspector::onRequest(const HttpRequest& req, HttpResponse* resp)
{
  if (req.path() == "/")
  {
    string result;
    MutexLockGuard lock(mutex_);
    // 遍历helps
    for (std::map<string, HelpList>::const_iterator helpListI = helps_.begin();
         helpListI != helps_.end();
         ++helpListI)
    {
      const HelpList& list = helpListI->second;
      for (HelpList::const_iterator it = list.begin();
           it != list.end();
           ++it)
      {
        result += "/";
        result += helpListI->first;      // module
        result += "/";
        result += it->first;         // command
        result += "\t";
        result += it->second;            // help
        result += "\n";
      }
    }
    resp->setStatusCode(HttpResponse::k200Ok);
    resp->setStatusMessage("OK");
    resp->setContentType("text/plain");
    resp->setBody(result);
  }
  else
  {
    // 以"/"进行分割，将得到的字符串保存在result中
    std::vector<string> result = split(req.path());
    // boost::split(result, req.path(), boost::is_any_of("/"));
    //std::copy(result.begin(), result.end(), std::ostream_iterator<string>(std::cout, ", "));
    //std::cout << "\n";
    bool ok = false;
    if (result.size() == 0)
    {
      // 这种情况是错误的，因此ok仍为false
    }
    else if (result.size() == 1)
    {
      // 只有module，没有command也是错的，因此ok仍为false
      string module = result[0];
    }
    else
    {
      string module = result[0];
      // 查找module所对应的命令列表
      std::map<string, CommandList>::const_iterator commListI = commands_.find(module);
      if (commListI != commands_.end())
      {
        string command = result[1];
        const CommandList& commList = commListI->second;
        // 查找command对应的命令
        CommandList::const_iterator it = commList.find(command);
        if (it != commList.end())
        {
          ArgList args(result.begin()+2, result.end());     // 传递给回调函数的参数表
          if (it->second)
          {
            resp->setStatusCode(HttpResponse::k200Ok);
            resp->setStatusMessage("OK");
            resp->setContentType("text/plain");
            const Callback& cb = it->second;
            resp->setBody(cb(req.method(), args));       // 调用cb将返回的字符串传给setBody
            ok = true;
          }
        }
      }
 
    }
 
    if (!ok)
    {
      resp->setStatusCode(HttpResponse::k404NotFound);
      resp->setStatusMessage("Not Found");
    }
    //resp->setCloseConnection(true);
  }
}
```

测试程序

``` c++
#include <muduo/net/inspect/Inspector.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/EventLoopThread.h>
 
using namespace muduo;
using namespace muduo::net;
 
int main()
{
  EventLoop loop;
  EventLoopThread t;    // 监控线程 ，这里“线程该没有真正创建，t.startLoop（）函数才是真正创建”
  /*Inspector 的loop 和main thread 的loop是不一样的，也就是说，现在已经有两个loop了*/
  Inspector ins(t.startLoop(), InetAddress(12345), "test");
  loop.loop();
}
```
在浏览器中输入127.0.0.1:12345/proc/status

``` c++
Name:   inspector_test
State:  S (sleeping)
Tgid:   2030
Pid:    2030
PPid:   1907
TracerPid:  0
Uid:    1000    1000    1000    1000
Gid:    1000    1000    1000    1000
FDSize: 256
Groups: 4 20 24 46 116 118 124 1000
VmPeak:    12348 kB
VmSize:    12348 kB
VmLck:         0 kB
VmHWM:      1580 kB
VmRSS:      1580 kB
VmData:     8408 kB
VmStk:       136 kB
VmExe:       860 kB
VmLib:      2868 kB
VmPTE:        32 kB
VmSwap:        0 kB
Threads:    2
SigQ:   0/9772
SigPnd: 0000000000000000
ShdPnd: 0000000000000000
SigBlk: 0000000000000000
SigIgn: 0000000000001000
SigCgt: 0000000180000000
CapInh: 0000000000000000
CapPrm: 0000000000000000
CapEff: 0000000000000000
CapBnd: ffffffffffffffff
Cpus_allowed:   3
Cpus_allowed_list:  0-1
Mems_allowed:   1
Mems_allowed_list:  0
voluntary_ctxt_switches:    22
nonvoluntary_ctxt_switches: 14
```
