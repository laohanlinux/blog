---
title: muduo simple example
date: 2014-07-01 01:22:24
tags:
- muduo
categories:
- muduo 
- network program
description: muduo 是一个linux 网络库，使用poll和epoll模式。 这些内容是本人在大学的时候，通过  << c++教程网 >>  所学的知识笔记. 已经过去好几年了，版本也是比较老的了，属于入门级别的教程； 如有错误，纯属正常!!!
---

## [42] 四个 服务器设计模型

五个简单TCP协议（MuduoManual.pdf P50）
muduo库网络模型使用示例（sudoku求解服务器MuduoManual.pdf P35 ）

- reactor（一个IO线程）
- reactor + threadpool (一个IO + 多个计算线程)
- multiple reactor （多个IO线程）
- one loop per thread + thread pool （多个IO线程 + 计算线程池）



网络编程关注4个半事件：

- 连接建立
- 连接断开
- 消息到达
- 信息发送完毕（对于低流量的服务来说，通常不需要关注该事件）

如何实现server

- 提供一个xxxServer类
-  在该类中包含一个TcpServer对象

注册一些事件

- OnConnection 
- OnMessage
- OnWriteComplete
- TcpConnection::shutdown（）内部实现，只关闭写入这一半

### 下面的程序都是用来解 数独 的，数独的实现如下

sudoku.h

``` c++
#ifndef MUDUO_EXAMPLES_SUDOKU_SUDOKU_H
#define MUDUO_EXAMPLES_SUDOKU_SUDOKU_H
 
 
#include <muduo/base/Types.h>
 
// FIXME, use (const char*, len) for saving memory copying.
muduo::string solveSudoku(const muduo::string& puzzle);
const int kCells = 81;
 
#endif
```
sudoku.cc

``` c++
#include "sudoku.h"
 
#include <vector>
#include <assert.h>
#include <string.h>
 
using namespace muduo;
 
struct Node;
typedef Node Column;
struct Node
{
    Node* left;
    Node* right;
    Node* up;
    Node* down;
    Column* col;
    int name;
    int size;
};
 
const int kMaxNodes = 1 + 81*4 + 9*9*9*4;
const int kMaxColumns = 400;
const int kRow = 100, kCol = 200, kBox = 300;
 
class SudokuSolver
{
 public:
    SudokuSolver(int board[kCells])
      : inout_(board),
        cur_node_(0)
    {
        stack_.reserve(100);
 
        root_ = new_column();
        root_->left = root_->right = root_;
        memset(columns_, 0, sizeof(columns_));
 
        bool rows[kCells][10] = { {false} };
        bool cols[kCells][10] = { {false} };
        bool boxes[kCells][10] = { {false} };
 
        for (int i = 0; i < kCells; ++i) {
            int row = i / 9;
            int col = i % 9;
            int box = row/3*3 + col/3;
            int val = inout_[i];
            rows[row][val] = true;
            cols[col][val] = true;
            boxes[box][val] = true;
        }
 
        for (int i = 0; i < kCells; ++i) {
            if (inout_[i] == 0) {
                append_column(i);
            }
        }
 
        for (int i = 0; i < 9; ++i) {
            for (int v = 1; v < 10; ++v) {
                if (!rows[i][v])
                    append_column(get_row_col(i, v));
                if (!cols[i][v])
                    append_column(get_col_col(i, v));
                if (!boxes[i][v])
                    append_column(get_box_col(i, v));
            }
        }
 
        for (int i = 0; i < kCells; ++i) {
            if (inout_[i] == 0) {
                int row = i / 9;
                int col = i % 9;
                int box = row/3*3 + col/3;
                //int val = inout[i];
                for (int v = 1; v < 10; ++v) {
                    if (!(rows[row][v] || cols[col][v] || boxes[box][v])) {
                        Node* n0 = new_row(i);
                        Node* nr = new_row(get_row_col(row, v));
                        Node* nc = new_row(get_col_col(col, v));
                        Node* nb = new_row(get_box_col(box, v));
                        put_left(n0, nr);
                        put_left(n0, nc);
                        put_left(n0, nb);
                    }
                }
            }
        }
    }
 
    bool solve()
    {
        if (root_->left == root_) {
            for (size_t i = 0; i < stack_.size(); ++i) {
                Node* n = stack_[i];
                int cell = -1;
                int val = -1;
                while (cell == -1 || val == -1) {
                    if (n->name < 100)
                        cell = n->name;
                    else
                        val = n->name % 10;
                    n = n->right;
                }
 
                //assert(cell != -1 && val != -1);
                inout_[cell] = val;
            }
            return true;
        }
 
        Column* const col = get_min_column();
        cover(col);
        for (Node* row = col->down; row != col; row = row->down) {
            stack_.push_back(row);
            for (Node* j = row->right; j != row; j = j->right) {
                cover(j->col);
            }
            if (solve()) {
                return true;
            }
            stack_.pop_back();
            for (Node* j = row->left; j != row; j = j->left) {
                uncover(j->col);
            }
        }
        uncover(col);
        return false;
    }
 
 private:
 
    Column* root_;
    int*    inout_;
    Column* columns_[400];
    std::vector<Node*> stack_;
    Node    nodes_[kMaxNodes];
    int     cur_node_;
 
    Column* new_column(int n = 0)
    {
        assert(cur_node_ < kMaxNodes);
        Column* c = &nodes_[cur_node_++];
        memset(c, 0, sizeof(Column));
        c->left = c;
        c->right = c;
        c->up = c;
        c->down = c;
        c->col = c;
        c->name = n;
        return c;
    }
 
    void append_column(int n)
    {
        assert(columns_[n] == NULL);
 
        Column* c = new_column(n);
        put_left(root_, c);
        columns_[n] = c;
    }
 
    Node* new_row(int col)
    {
        assert(columns_[col] != NULL);
        assert(cur_node_ < kMaxNodes);
 
        Node* r = &nodes_[cur_node_++];
 
        //Node* r = new Node;
        memset(r, 0, sizeof(Node));
        r->left = r;
        r->right = r;
        r->up = r;
        r->down = r;
        r->name = col;
        r->col = columns_[col];
        put_up(r->col, r);
        return r;
    }
 
    int get_row_col(int row, int val)
    {
        return kRow+row*10+val;
    }
 
    int get_col_col(int col, int val)
    {
        return kCol+col*10+val;
    }
 
    int get_box_col(int box, int val)
    {
        return kBox+box*10+val;
    }
 
    Column* get_min_column()
    {
        Column* c = root_->right;
        int min_size = c->size;
        if (min_size > 1) {
            for (Column* cc = c->right; cc != root_; cc = cc->right) {
                if (min_size > cc->size) {
                    c = cc;
                    min_size = cc->size;
                    if (min_size <= 1)
                        break;
                }
            }
        }
        return c;
    }
 
    void cover(Column* c)
    {
        c->right->left = c->left;
        c->left->right = c->right;
        for (Node* row = c->down; row != c; row = row->down) {
            for (Node* j = row->right; j != row; j = j->right) {
                j->down->up = j->up;
                j->up->down = j->down;
                j->col->size--;
            }
        }
    }
 
    void uncover(Column* c)
    {
        for (Node* row = c->up; row != c; row = row->up) {
            for (Node* j = row->left; j != row; j = j->left) {
                j->col->size++;
                j->down->up = j;
                j->up->down = j;
            }
        }
        c->right->left = c;
        c->left->right = c;
    }
 
    void put_left(Column* old, Column* nnew)
    {
        nnew->left = old->left;
        nnew->right = old;
        old->left->right = nnew;
        old->left = nnew;
    }
 
    void put_up(Column* old, Node* nnew)
    {
        nnew->up = old->up;
        nnew->down = old;
        old->up->down = nnew;
        old->up = nnew;
        old->size++;
        nnew->col = old;
    }
};
 
string solveSudoku(const string& puzzle)
{
  assert(puzzle.size() == implicit_cast<size_t>(kCells));
 
  string result = "NoSolution";
 
  int board[kCells] = { 0 };
  bool valid = true;
  for (int i = 0; i < kCells; ++i)
  {
    board[i] = puzzle[i] - '0';
    valid = valid && (0 <= board[i] && board[i] <= 9);
  }
 
  if (valid)
  {
    SudokuSolver s(board);
    if (s.solve())
    {
      result.clear();
      result.resize(kCells);
      for (int i = 0; i < kCells; ++i)
      {
        result[i] = static_cast<char>(board[i] + '0');
      }
    }
  }
  return result;
}
```

### reactor模型

只有一个IO线程：这个IO线程既负责listenfd也负责connfd

``` c++
#include "sudoku.h"
 
#include <muduo/base/Atomic.h>
#include <muduo/base/Logging.h>
#include <muduo/base/Thread.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
#include <muduo/net/TcpServer.h>
 
#include <boost/bind.hpp>
 
#include <utility>
 
#include <mcheck.h>
#include <stdio.h>
#include <unistd.h>
 
using namespace muduo;
using namespace muduo::net;
 
class SudokuServer
{
 public:
  SudokuServer(EventLoop* loop, const InetAddress& listenAddr)
    : loop_(loop),
      server_(loop, listenAddr, "SudokuServer"),
      startTime_(Timestamp::now())
  {
    server_.setConnectionCallback(
        boost::bind(&SudokuServer::onConnection, this, _1));
    server_.setMessageCallback(
        boost::bind(&SudokuServer::onMessage, this, _1, _2, _3));
  }
 
  void start()
  {
    server_.start();
  }
 
 private:
  void onConnection(const TcpConnectionPtr& conn)
  {
    LOG_TRACE << conn->peerAddress().toIpPort() << " -> "
        << conn->localAddress().toIpPort() << " is "
        << (conn->connected() ? "UP" : "DOWN");
  }
 
  void onMessage(const TcpConnectionPtr& conn, Buffer* buf, Timestamp)
  {
    LOG_DEBUG << conn->name();
    size_t len = buf->readableBytes();
    while (len >= kCells + 2)
    {
      const char* crlf = buf->findCRLF();
      if (crlf)
      {
        string request(buf->peek(), crlf);
        buf->retrieveUntil(crlf + 2);
        len = buf->readableBytes();
        if (!processRequest(conn, request))
        {
          conn->send("Bad Request!\r\n");
          conn->shutdown();
          break;
        }
      }
      else if (len > 100) // id + ":" + kCells + "\r\n"
      {
        conn->send("Id too long!\r\n");
        conn->shutdown();
        break;
      }
      else
      {
        break;
      }
    }
  }
 
  bool processRequest(const TcpConnectionPtr& conn, const string& request)
  {
    string id;
    string puzzle;
    bool goodRequest = true;
 
    string::const_iterator colon = find(request.begin(), request.end(), ':');
    if (colon != request.end())
    {
      id.assign(request.begin(), colon);
      puzzle.assign(colon+1, request.end());
    }
    else
    {
      puzzle = request;
    }
 
    if (puzzle.size() == implicit_cast<size_t>(kCells))
    {
      LOG_DEBUG << conn->name();
      string result = solveSudoku(puzzle);
      if (id.empty())
      {
        conn->send(result+"\r\n");
      }
      else
      {
        conn->send(id+":"+result+"\r\n");
      }
    }
    else
    {
      goodRequest = false;
    }
    return goodRequest;
  }
 
  EventLoop* loop_;
  TcpServer server_;
  Timestamp startTime_;
};
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid() << ", tid = " << CurrentThread::tid();
  EventLoop loop;
  InetAddress listenAddr(9981);
  SudokuServer server(&loop, listenAddr);
 
  server.start();
 
  loop.loop();
}
```
### multiple reactor

- IO线程的数目多个
- EventLoopThreadPoll IO线程池
- 直接设置server_.setThreadNum(numThreads)就OK了

main reactor 负责listenfd accept , sub reactor  负责connfd, 使用roundbin轮叫策略
来一个连接，就选择下一个EventLoop，这样让多个连接分配给若干个EventLoop来处理，
而每个EventLoop属于一个IO线程，也就意味着，多个连接分配给若干个IO线程来处理。

``` c++
include "sudoku.h"
 
#include <muduo/base/Atomic.h>
#include <muduo/base/Logging.h>
#include <muduo/base/Thread.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
#include <muduo/net/TcpServer.h>
 
#include <boost/bind.hpp>
 
#include <utility>
 
#include <mcheck.h>
#include <stdio.h>
#include <unistd.h>
 
using namespace muduo;
using namespace muduo::net;
 
class SudokuServer
{
 public:
  SudokuServer(EventLoop* loop, const InetAddress& listenAddr, int numThreads)
    : loop_(loop),
      server_(loop, listenAddr, "SudokuServer"),
      numThreads_(numThreads),
      startTime_(Timestamp::now())
  {
    server_.setConnectionCallback(
        boost::bind(&SudokuServer::onConnection, this, _1));
    server_.setMessageCallback(
        boost::bind(&SudokuServer::onMessage, this, _1, _2, _3));
    server_.setThreadNum(numThreads);
  }
 
  void start()
  {
    LOG_INFO << "starting " << numThreads_ << " threads.";
    server_.start();
  }
 
 private:
  void onConnection(const TcpConnectionPtr& conn)
  {
    LOG_TRACE << conn->peerAddress().toIpPort() << " -> "
        << conn->localAddress().toIpPort() << " is "
        << (conn->connected() ? "UP" : "DOWN");
  }
 
  void onMessage(const TcpConnectionPtr& conn, Buffer* buf, Timestamp)
  {
    LOG_DEBUG << conn->name();
    size_t len = buf->readableBytes();
    while (len >= kCells + 2)
    {
      const char* crlf = buf->findCRLF();
      if (crlf)
      {
        string request(buf->peek(), crlf);
        buf->retrieveUntil(crlf + 2);
        len = buf->readableBytes();
        if (!processRequest(conn, request))
        {
          conn->send("Bad Request!\r\n");
          conn->shutdown();
          break;
        }
      }
      else if (len > 100) // id + ":" + kCells + "\r\n"
      {
        conn->send("Id too long!\r\n");
        conn->shutdown();
        break;
      }
      else
      {
        break;
      }
    }
  }
 
  bool processRequest(const TcpConnectionPtr& conn, const string& request)
  {
    string id;
    string puzzle;
    bool goodRequest = true;
 
    string::const_iterator colon = find(request.begin(), request.end(), ':');
    if (colon != request.end())
    {
      id.assign(request.begin(), colon);
      puzzle.assign(colon+1, request.end());
    }
    else
    {
      puzzle = request;
    }
 
    if (puzzle.size() == implicit_cast<size_t>(kCells))
    {
      LOG_DEBUG << conn->name();
      string result = solveSudoku(puzzle);
      if (id.empty())
      {
        conn->send(result+"\r\n");
      }
      else
      {
        conn->send(id+":"+result+"\r\n");
      }
    }
    else
    {
      goodRequest = false;
    }
    return goodRequest;
  }
 
  EventLoop* loop_;
  TcpServer server_;
  int numThreads_;
  Timestamp startTime_;
};
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid() << ", tid = " << CurrentThread::tid();
  int numThreads = 0;
  if (argc > 1)
  {
    numThreads = atoi(argv[1]);
  }
  EventLoop loop;
  InetAddress listenAddr(9981);
  SudokuServer server(&loop, listenAddr, numThreads);
 
  server.start();
 
  loop.loop();
}
```

### reactor + threadPool

- 一个IO线程，多个计算thread的模式
- EventLoop + threadpool + numThreads_

``` c++
include "sudoku.h"
 
#include <muduo/base/Atomic.h>
#include <muduo/base/Logging.h>
#include <muduo/base/Thread.h>
#include <muduo/base/ThreadPool.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
#include <muduo/net/TcpServer.h>
 
#include <boost/bind.hpp>
 
#include <utility>
 
#include <mcheck.h>
#include <stdio.h>
#include <unistd.h>
 
using namespace muduo;
using namespace muduo::net;
 
class SudokuServer
{
 public:
  SudokuServer(EventLoop* loop, const InetAddress& listenAddr, int numThreads)
    : loop_(loop),
      server_(loop, listenAddr, "SudokuServer"),
      numThreads_(numThreads),
      startTime_(Timestamp::now())
  {
    server_.setConnectionCallback(
        boost::bind(&SudokuServer::onConnection, this, _1));
    server_.setMessageCallback(
        boost::bind(&SudokuServer::onMessage, this, _1, _2, _3));
  }
 
  void start()
  {
    LOG_INFO << "starting " << numThreads_ << " threads.";
    threadPool_.start(numThreads_);//线程池的线程个数
    server_.start();
  }
 
 private:
  void onConnection(const TcpConnectionPtr& conn)
  {
    LOG_TRACE << conn->peerAddress().toIpPort() << " -> "
        << conn->localAddress().toIpPort() << " is "
        << (conn->connected() ? "UP" : "DOWN");
  }
 
  void onMessage(const TcpConnectionPtr& conn, Buffer* buf, Timestamp)
  {
    LOG_DEBUG << conn->name();
    size_t len = buf->readableBytes();
    while (len >= kCells + 2)
    {
      const char* crlf = buf->findCRLF();
      if (crlf)
      {
        string request(buf->peek(), crlf);
        buf->retrieveUntil(crlf + 2);
        len = buf->readableBytes();
        if (!processRequest(conn, request))
        {
          conn->send("Bad Request!\r\n");
          conn->shutdown();
          break;
        }
      }
      else if (len > 100) // id + ":" + kCells + "\r\n"
      {
        conn->send("Id too long!\r\n");
        conn->shutdown();
        break;
      }
      else
      {
        break;
      }
    }
  }
 
  bool processRequest(const TcpConnectionPtr& conn, const string& request)
  {
    string id;
    string puzzle;
    bool goodRequest = true;
 
    string::const_iterator colon = find(request.begin(), request.end(), ':');
    if (colon != request.end())
    {
      id.assign(request.begin(), colon);
      puzzle.assign(colon+1, request.end());
    }
    else
    {
      puzzle = request;
    }
    /*计算线程中的线程进行处理*/
    if (puzzle.size() == implicit_cast<size_t>(kCells))
    {
      threadPool_.run(boost::bind(&solve, conn, puzzle, id));
    }
    else
    {
      goodRequest = false;
    }
    return goodRequest;
  }
 
  static void solve(const TcpConnectionPtr& conn,
                    const string& puzzle,
                    const string& id)
  {
    LOG_DEBUG << conn->name();
    string result = solveSudoku(puzzle);
    /*这里处理完数据后，conn->send() 还是在IO线程中发送，而不是
在计算线程中处理
    */
    if (id.empty())
    {
      conn->send(result+"\r\n");
    }
    else
    {
      conn->send(id+":"+result+"\r\n");
    }
  }
 
  EventLoop* loop_;
  TcpServer server_;
  ThreadPool threadPool_; //计算线程池
  int numThreads_;
  Timestamp startTime_;
};
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid() << ", tid = " << CurrentThread::tid();
  int numThreads = 0;
  if (argc > 1)
  {
    numThreads = atoi(argv[1]);
  }
  EventLoop loop;
  InetAddress listenAddr(9981);
  SudokuServer server(&loop, listenAddr, numThreads);
 
  server.start();
 
  loop.loop();
}
```

### multiple reactors+threadpool

EventLoopThreadPoll + threadpool + IOnumThreads_ + ThreadPoolnumThreads_

<center>![](http://laohanlinux.github.io/images/img/blog/muduo-simple-example/multiplereactorthreadpool.png)</center>

``` c++
include "sudoku.h"
 
#include <muduo/base/Atomic.h>
#include <muduo/base/Logging.h>
#include <muduo/base/Thread.h>
#include <muduo/base/ThreadPool.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/InetAddress.h>
#include <muduo/net/TcpServer.h>
 
#include <boost/bind.hpp>
 
#include <utility>
 
#include <mcheck.h>
#include <stdio.h>
#include <unistd.h>
 
using namespace muduo;
using namespace muduo::net;
 
class SudokuServer
{
 public:
  SudokuServer(EventLoop* loop, const InetAddress& listenAddr, int numThreads)
    : loop_(loop),
      server_(loop, listenAddr, "SudokuServer"),
      numThreads_(numThreads),
      startTime_(Timestamp::now())
  {
    server_.setConnectionCallback(
        boost::bind(&SudokuServer::onConnection, this, _1));
    server_.setMessageCallback(
        boost::bind(&SudokuServer::onMessage, this, _1, _2, _3));
     server_.setThreadNum(numThreads);//IO线程池的初始化
  }
 
  void start()
  {
    LOG_INFO << "starting " << numThreads_ << " threads.";
    threadPool_.start(numThreads_);
    server_.start();
  }
 
 private:
  void onConnection(const TcpConnectionPtr& conn)
  {
    LOG_TRACE << conn->peerAddress().toIpPort() << " -> "
        << conn->localAddress().toIpPort() << " is "
        << (conn->connected() ? "UP" : "DOWN");
  }
 
  void onMessage(const TcpConnectionPtr& conn, Buffer* buf, Timestamp)
  {
    LOG_DEBUG << conn->name();
    size_t len = buf->readableBytes();
    while (len >= kCells + 2)
    {
      const char* crlf = buf->findCRLF();
      if (crlf)
      {
        string request(buf->peek(), crlf);
        buf->retrieveUntil(crlf + 2);
        len = buf->readableBytes();
        if (!processRequest(conn, request))
        {
          conn->send("Bad Request!\r\n");
          conn->shutdown();
          break;
        }
      }
      else if (len > 100) // id + ":" + kCells + "\r\n"
      {
        conn->send("Id too long!\r\n");
        conn->shutdown();
        break;
      }
      else
      {
        break;
      }
    }
  }
 
  bool processRequest(const TcpConnectionPtr& conn, const string& request)
  {
    string id;
    string puzzle;
    bool goodRequest = true;
 
    string::const_iterator colon = find(request.begin(), request.end(), ':');
    if (colon != request.end())
    {
      id.assign(request.begin(), colon);
      puzzle.assign(colon+1, request.end());
    }
    else
    {
      puzzle = request;
    }
    /*计算线程中的线程进行处理*/
    if (puzzle.size() == implicit_cast<size_t>(kCells))
    {
      threadPool_.run(boost::bind(&solve, conn, puzzle, id));
    }
    else
    {
      goodRequest = false;
    }
    return goodRequest;
  }
 
  static void solve(const TcpConnectionPtr& conn,
                    const string& puzzle,
                    const string& id)
  {
    LOG_DEBUG << conn->name();
    string result = solveSudoku(puzzle);
    /*这里处理完数据后，conn->send() 还是在IO线程中发送，而不是
在计算线程中处理
    */
    if (id.empty())
    {
      conn->send(result+"\r\n");
    }
    else
    {
      conn->send(id+":"+result+"\r\n");
    }
  }
 
  EventLoop* loop_;
  TcpServer server_;
  ThreadPool threadPool_;
  int numThreads_;
  Timestamp startTime_;
};
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid() << ", tid = " << CurrentThread::tid();
  int numThreads = 0;
  if (argc > 1)
  {
    numThreads = atoi(argv[1]);
  }
  EventLoop loop;
  InetAddress listenAddr(9981);
  SudokuServer server(&loop, listenAddr, numThreads);
 
  server.start();
 
  loop.loop();
}
```
sudoKu 求解服务器，既是一个IO密集型，又是一个计算密集型的服务

IO线程池 + 计算线程池
计算时间如果比较久，就会使得IO线程阻塞，IO线程很快就用尽，就不处理大量的并发连接
一个IO线程+计算线程池
使用muduo 库来编程还是比较容易的，有兴趣的朋友可以试一下！

## [43] 文件传输服务器

- 文件传输（MuduoManual.pdf P57）
- examples/filetransfer/download.cc
- examples/filetransfer/download2.cc
- examples/filetransfer/download3.cc

tests/Filetransfer_test.cc

### 单线程模式之一次性发完一个文件

``` c++
#include <muduo/base/Logging.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/TcpServer.h>
 
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
const char* g_file = NULL;
 
// FIXME: use FileUtil::readFile()
string readFile(const char* filename)
{
  string content;
  FILE* fp = ::fopen(filename, "rb");
  if (fp)
  {
    // inefficient!!!
    const int kBufSize = 1024*1024;
    char iobuf[kBufSize];
    ::setbuffer(fp, iobuf, sizeof iobuf);
 
    char buf[kBufSize];
    size_t nread = 0;
    while ( (nread = ::fread(buf, 1, sizeof buf, fp)) > 0)
    {
      content.append(buf, nread);
    }
    ::fclose(fp);
  }
  return content;
}
 
void onHighWaterMark(const TcpConnectionPtr& conn, size_t len)
{
  LOG_INFO << "HighWaterMark " << len;
}
 
void onConnection(const TcpConnectionPtr& conn)
{
  LOG_INFO << "FileServer - " << conn->peerAddress().toIpPort() << " -> "
           << conn->localAddress().toIpPort() << " is "
           << (conn->connected() ? "UP" : "DOWN");
  if (conn->connected())
  {
    LOG_INFO << "FileServer - Sending file " << g_file
             << " to " << conn->peerAddress().toIpPort();
    conn->setHighWaterMarkCallback(onHighWaterMark, 64*1024);
    string fileContent = readFile(g_file);
    conn->send(fileContent);
    conn->shutdown();
    LOG_INFO << "FileServer - done";
  }
}
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid();
  if (argc > 1)
  {
    g_file = argv[1];
 
    EventLoop loop;
    InetAddress listenAddr(2021);
    TcpServer server(&loop, listenAddr, "FileServer");
    server.setConnectionCallback(onConnection);
    server.start();
    loop.loop();
  }
  else
  {
    fprintf(stderr, "Usage: %s file_for_downloading\n", argv[0]);
  }
}
```

### 单线程模型之分块发送文件

``` c++
#include <muduo/base/Logging.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/TcpServer.h>
 
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
void onHighWaterMark(const TcpConnectionPtr& conn, size_t len)
{
  LOG_INFO << "HighWaterMark " << len;
}
 
const int kBufSize = 64*1024;
const char* g_file = NULL;
 
void onConnection(const TcpConnectionPtr& conn)
{
  LOG_INFO << "FileServer - " << conn->peerAddress().toIpPort() << " -> "
           << conn->localAddress().toIpPort() << " is "
           << (conn->connected() ? "UP" : "DOWN");
  if (conn->connected())
  {
    LOG_INFO << "FileServer - Sending file " << g_file
             << " to " << conn->peerAddress().toIpPort();
    /*高水位标志的回调函数*/
    conn->setHighWaterMarkCallback(onHighWaterMark, kBufSize+1);
 
    FILE* fp = ::fopen(g_file, "rb");
    if (fp)
    {
    /*设置conn 的上下文*/
      conn->setContext(fp);
      char buf[kBufSize];
      size_t nread = ::fread(buf, 1, sizeof buf, fp);
      conn->send(buf, nread);
    }
    /*发送完毕就shutdown connection*/
    else
    {
      conn->shutdown();
      LOG_INFO << "FileServer - no such file";
    }
  }
  /*如果连接关闭，那么就关闭文件指针*/
  else
  {
    if (!conn->getContext().empty())
    {
      FILE* fp = boost::any_cast<FILE*>(conn->getContext());
      if (fp)
      {
        ::fclose(fp);
      }
    }
  }
}
 
/*如果发完一块，还有其他块，那么接着发送，这只fp是保存在
 
connection的上下文中，所以是同一个文件指针*/
void onWriteComplete(const TcpConnectionPtr& conn)
{
  FILE* fp = boost::any_cast<FILE*>(conn->getContext());
  char buf[kBufSize];
  size_t nread = ::fread(buf, 1, sizeof buf, fp);
  if (nread > 0)
  {
    conn->send(buf, nread);
  }
  /*如果发完也关闭掉*/
  else
  {
    ::fclose(fp);
    fp = NULL;
    conn->setContext(fp);
    conn->shutdown();
    LOG_INFO << "FileServer - done";
  }
}
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid();
  if (argc > 1)
  {
    g_file = argv[1];
 
    EventLoop loop;
    InetAddress listenAddr(2021);
    TcpServer server(&loop, listenAddr, "FileServer");
    server.setConnectionCallback(onConnection);
    server.setWriteCompleteCallback(onWriteComplete);
    server.start();
    loop.loop();
  }
  else
  {
    fprintf(stderr, "Usage: %s file_for_downloading\n", argv[0]);
  }
}
```

### 单线程模型之分块发送（智能指针）

``` c++
#include <muduo/base/Logging.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/TcpServer.h>
 
#include <boost/shared_ptr.hpp>
 
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
/*这个程序和第二个是一样，只是这里的文件指针是智能共享的，
不用我们手动关闭*/
 
void onHighWaterMark(const TcpConnectionPtr& conn, size_t len)
{
  LOG_INFO << "HighWaterMark " << len;
}
 
const int kBufSize = 64*1024;
const char* g_file = NULL;
typedef boost::shared_ptr<FILE> FilePtr;
 
void onConnection(const TcpConnectionPtr& conn)
{
  LOG_INFO << "FileServer - " << conn->peerAddress().toIpPort() << " -> "
           << conn->localAddress().toIpPort() << " is "
           << (conn->connected() ? "UP" : "DOWN");
  if (conn->connected())
  {
    LOG_INFO << "FileServer - Sending file " << g_file
             << " to " << conn->peerAddress().toIpPort();
    conn->setHighWaterMarkCallback(onHighWaterMark, kBufSize+1);
 
    FILE* fp = ::fopen(g_file, "rb");
    if (fp)
    {
    /*这里ctx接受两个参数，因为ctx不是类的指针，所以他不是调用delete
    来消费ctx指针，而是调用fclose这个函数来消费这个ctx指针*/
      FilePtr ctx(fp, ::fclose);
      conn->setContext(ctx);
      char buf[kBufSize];
      size_t nread = ::fread(buf, 1, sizeof buf, fp);
      conn->send(buf, nread);
    }
    else
    {
      conn->shutdown();
      LOG_INFO << "FileServer - no such file";
    }
  }
}
 
void onWriteComplete(const TcpConnectionPtr& conn)
{
  const FilePtr& fp = boost::any_cast<const FilePtr&>(conn->getContext());
  char buf[kBufSize];
  size_t nread = ::fread(buf, 1, sizeof buf, get_pointer(fp));
  if (nread > 0)
  {
    conn->send(buf, nread);
  }
  else
  {
    conn->shutdown();
    LOG_INFO << "FileServer - done";
  }
}
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid();
  if (argc > 1)
  {
    g_file = argv[1];
 
    EventLoop loop;
    InetAddress listenAddr(2021);
    TcpServer server(&loop, listenAddr, "FileServer");
    server.setConnectionCallback(onConnection);
    server.setWriteCompleteCallback(onWriteComplete);
    server.start();
    loop.loop();
  }
  else
  {
    fprintf(stderr, "Usage: %s file_for_downloading\n", argv[0]);
  }
}
```

## [44-45] muduo实现简单了聊天功能

### 聊天服务器（MuduoManual.pdf P66）

- examples/asio/chat/server.cc 单线程 
- examples/asio/chat/server_threaded.cc，多线程TcpServer，并用mutex来保护共享数据 
- examples/asio/chat/server_threaded_efficient.cc,借shared_ptr实现copy-on-write的手法来降低锁竞争 
- examples/asio/chat/server_threaded_highperformance.cc，采用thread local变量实现多线程高效转发 

消息分为包头与包体，每条消息有一个4字节的头部，以网络序存放字符串的长度。包体是一个字符串，字符串也不一定以’\0’结尾。比方说有两条消息"hello"和"chenshuo"，那么打包后的字节流是： `0x00,0x00,0x00,0x05, 'h','e','l','l','o',0x00,0x00,0x00,0x08,'c','h', 'e','n','s','h','u','o'` 共21字节。 

<center> ![](hhttp://laohanlinux.github.io/images/img/blog/muduo-simple-example/chat1.png)</center>

`shared_ptr` 指针

借`shared_ptr`实现`copy on write`

`shared_ptr`是引用计数智能指针，如果当前只有一个观察者，那么引用计数为`1`,可以用`shared_ptr::unique()`来判断 对于`write`端，如果发现引用计数为`1`，这时可以安全地修改对象，不必担心有人在读它。 对于`read`端，在读之前把引用计数加`1`，读完之后减`1`，这样可以保证在读的期间其引用计数大于`1`，可以阻止并发写。 比较难的是，对于`write`端，如果发现引用计数大于`1`，该如何处理?既然要更新数据，肯定要加锁，如果这时候其他线程正在读，那么不能在原来的数据上修改，得创建一个副本，在副本上修改，修改完了再替换。如果没有用户在读，那么可以直接修改。 

### code.h

``` c++
#ifndef MUDUO_EXAMPLES_ASIO_CHAT_CODEC_H
#define MUDUO_EXAMPLES_ASIO_CHAT_CODEC_H
 
#include <muduo/base/Logging.h>
#include <muduo/net/Buffer.h>
#include <muduo/net/Endian.h>
#include <muduo/net/TcpConnection.h>
 
#include <boost/function.hpp>
#include <boost/noncopyable.hpp>
 
class LengthHeaderCodec : boost::noncopyable
{
 public:
  typedef boost::function<void (const muduo::net::TcpConnectionPtr&,
                                const muduo::string& message,
                                muduo::Timestamp)> StringMessageCallback;
 
  explicit LengthHeaderCodec(const StringMessageCallback& cb)
    : messageCallback_(cb)
  {
  }
/*消息到达的回调函数*/
  void onMessage(const muduo::net::TcpConnectionPtr& conn,
                 muduo::net::Buffer* buf,
                 muduo::Timestamp receiveTime)
  {
  /*这里可能有多条信息一起到达*/
    while (buf->readableBytes() >= kHeaderLen) // kHeaderLen == 4
    {
      // FIXME: use Buffer::peekInt32()
        /*这里的消息包括消息头(包头)和消息尾(包体)*/
      const void* data = buf->peek(); //这里只是查看一下数据而已，并没有取出数据
      /*读出的是对方发过来的网络字节序(大端)的前4个字节(header)*/
      int32_t be32 = *static_cast<const int32_t*>(data); // SIGBUS
        /*把网络字节转为主机字节序*/
      const int32_t len = muduo::net::sockets::networkToHost32(be32);
        /*这里假设消息的包体长度不超过64k */
      if (len > 65536 || len < 0) //消息不合法
      {
        LOG_ERROR << "Invalid length " << len;
        conn->shutdown();  // FIXME: disable reading
        break;
      }
 
      else if (buf->readableBytes() >= len + kHeaderLen)
      {
        buf->retrieve(kHeaderLen);
        /*这里还没有取出消息的包体，只是peek一下*/
        muduo::string message(buf->peek(), len);
        /*回调应用程序，让应用层来处理包体*/
        messageCallback_(conn, message, receiveTime);
        /*取出包体*/
        buf->retrieve(len);
      }
      /*未达到完整的一条消息*/
      else
      {
        break;
      }
    }
  }
 
  // FIXME: TcpConnectionPtr
    /*编码函数*/
  void send(muduo::net::TcpConnection* conn,
            const muduo::StringPiece& message)
  {
    muduo::net::Buffer buf;
    buf.append(message.data(), message.size());
    int32_t len = static_cast<int32_t>(message.size());
    int32_t be32 = muduo::net::sockets::hostToNetwork32(len);
    buf.prepend(&be32, sizeof be32);
    /*编完码后,发送出去*/
    conn->send(&buf);
  }
 
 private:
  StringMessageCallback messageCallback_;
  const static size_t kHeaderLen = sizeof(int32_t);
};
```

### examples/asio/chat/server.cc 单线程

``` c++
#include "codec.h"
 
#include <muduo/base/Logging.h>
#include <muduo/base/Mutex.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/TcpServer.h>
 
#include <boost/bind.hpp>
 
#include <set>
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
/*
Program :这是一个单线程的程序，不需要mutex
 
*/
 
class ChatServer : boost::noncopyable
{
 public:
  ChatServer(EventLoop* loop,
             const InetAddress& listenAddr)
  : loop_(loop),
    server_(loop, listenAddr, "ChatServer"),
    /*消息编解码初始化，邋onString錗essage()为编解码完后的回调函数*/
    codec_(boost::bind(&ChatServer::onStringMessage, this, _1, _2, _3))
  {
    server_.setConnectionCallback(
        boost::bind(&ChatServer::onConnection, this, _1));
    /*消息达到时的回调函数*/
    server_.setMessageCallback(
        boost::bind(&LengthHeaderCodec::onMessage, &codec_, _1, _2, _3));
  }
 
  void start()
  {
    server_.start();
  }
 
 private:
    /*只有一个IO线程，因而这里的connection_不需要mutex保护*/
    /*连接到达对等方对开连接时的回调函数*/
  void onConnection(const TcpConnectionPtr& conn)
  {
    LOG_INFO << conn->localAddress().toIpPort() << " -> "
             << conn->peerAddress().toIpPort() << " is "
             << (conn->connected() ? "UP" : "DOWN");
    /*如果已经连接了，回调*/
    if (conn->connected())
    {
      connections_.insert(conn);
    }
    /*连接断开*/
    else
    {
      connections_.erase(conn);
    }
  }
/*编解码class 的回调函数*/
/*转发消息给所有客户端*/
  void onStringMessage(const TcpConnectionPtr&,
                       const string& message,
                       Timestamp)
  {
  /*只有一个IO线程，因而这里的connections_不需要mutex保护;
    转发信息给所有客户端
  */
    for (ConnectionList::iterator it = connections_.begin();
        it != connections_.end();
        ++it)
    {
      codec_.send(get_pointer(*it), message);
    }
  }
 
  typedef std::set<TcpConnectionPtr> ConnectionList;
  EventLoop* loop_;
  TcpServer server_;
  /*消息编解码class*/
  LengthHeaderCodec codec_;
  /*连接列表*/
  ConnectionList connections_;
};
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid();
  if (argc > 1)
  {
    EventLoop loop;
    uint16_t port = static_cast<uint16_t>(atoi(argv[1]));
    InetAddress serverAddr(port);
    ChatServer server(&loop, serverAddr);
    server.start();
    loop.loop();
  }
  else
  {
    printf("Usage: %s port\n", argv[0]);
  }
}
```

### examples/asio/chat/server_threaded.cc，多线程TcpServer，并用mutex来保护共享数据

``` c++
#include "codec.h"
 
#include <muduo/base/Logging.h>
#include <muduo/base/Mutex.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/TcpServer.h>
 
#include <boost/bind.hpp>
 
#include <set>
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
/*这是一个典型的多线程聊天程序,multipleReactor 模型*/
 
class ChatServer : boost::noncopyable
{
 public:
  ChatServer(EventLoop* loop,
             const InetAddress& listenAddr)
  : loop_(loop),
    server_(loop, listenAddr, "ChatServer"),
    codec_(boost::bind(&ChatServer::onStringMessage, this, _1, _2, _3))
  {
    server_.setConnectionCallback(
        boost::bind(&ChatServer::onConnection, this, _1));
    server_.setMessageCallback(
        boost::bind(&LengthHeaderCodec::onMessage, &codec_, _1, _2, _3));
  }
 
  void setThreadNum(int numThreads)
  {
    server_.setThreadNum(numThreads);
  }
 
  void start()
  {
    server_.start();
  }
 
 private:
  void onConnection(const TcpConnectionPtr& conn)
  {
    LOG_INFO << conn->localAddress().toIpPort() << " -> "
        << conn->peerAddress().toIpPort() << " is "
        << (conn->connected() ? "UP" : "DOWN");
 
    MutexLockGuard lock(mutex_);
    if (conn->connected())
    {
      connections_.insert(conn);
    }
    else
    {
      connections_.erase(conn);
    }
  }
 
  void onStringMessage(const TcpConnectionPtr&,
                       const string& message,
                       Timestamp)
  {
  /*多线程需要保护连接列表*/
    MutexLockGuard lock(mutex_);
    for (ConnectionList::iterator it = connections_.begin();
        it != connections_.end();
        ++it)
    {
      codec_.send(get_pointer(*it), message);
    }
  }
 
  typedef std::set<TcpConnectionPtr> ConnectionList;
  EventLoop* loop_;
  TcpServer server_;
  LengthHeaderCodec codec_;
  MutexLock mutex_;
  ConnectionList connections_;
};
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid();
  if (argc > 1)
  {
    EventLoop loop;
    uint16_t port = static_cast<uint16_t>(atoi(argv[1]));
    InetAddress serverAddr(port);
    ChatServer server(&loop, serverAddr);
    if (argc > 2)
    {
      server.setThreadNum(atoi(argv[2]));
    }
    server.start();
    loop.loop();
  }
  else
  {
    printf("Usage: %s port [thread_num]\n", argv[0]);
  }
}
```

### examples/asio/chat/server_threaded_efficient.cc,借shared_ptr实现copy-on-write的手法来降低锁竞争

``` c++
#include "codec.h"
 
#include <muduo/base/Logging.h>
#include <muduo/base/Mutex.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/TcpServer.h>
 
#include <boost/bind.hpp>
#include <boost/shared_ptr.hpp>
 
#include <set>
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
/*这是一个典型的多线程聊天程序multipleReactor 模型，
但是这里使用了一些编程技巧，达到一些优化*/
 
class ChatServer : boost::noncopyable
{
 public:
  ChatServer(EventLoop* loop,
             const InetAddress& listenAddr)
  : loop_(loop),
    server_(loop, listenAddr, "ChatServer"),//loop : acceptor loop
    codec_(boost::bind(&ChatServer::onStringMessage, this, _1, _2, _3)),
    connections_(new ConnectionList)//初始化时，share_ptr的引用为1
  {
    server_.setConnectionCallback(
        boost::bind(&ChatServer::onConnection, this, _1));
    server_.setMessageCallback(
        boost::bind(&LengthHeaderCodec::onMessage, &codec_, _1, _2, _3));
  }
 
  void setThreadNum(int numThreads)
  {
    server_.setThreadNum(numThreads);
  }
 
  void start()
  {
    server_.start();
  }
 
 private:
  void onConnection(const TcpConnectionPtr& conn)
  {
    LOG_INFO << conn->localAddress().toIpPort() << " -> "
        << conn->peerAddress().toIpPort() << " is "
        << (conn->connected() ? "UP" : "DOWN");
 
    MutexLockGuard lock(mutex_);
    if (!connections_.unique())//说明引用计数大于1
    {//new ConnectionList(*connections_) 这段代码拷贝了一份ConnectionList
    //connections_原来的引用计数减1，而connections_现在的引用计数
    // 等于1
      connections_.reset(new ConnectionList(*connections_));
    }
    //所以这里断言才会成功
    assert(connections_.unique());
    /*在复本上修改,不会影响读者,所以读者在遍历列表的时候,
    不需要mutex保护*/
    if (conn->connected())
    {
      connections_->insert(conn);
    }
    else
    {
      connections_->erase(conn);
    }
  }
 
  typedef std::set<TcpConnectionPtr> ConnectionList;
  typedef boost::shared_ptr<ConnectionList> ConnectionListPtr;
/*读操作*/
  void onStringMessage(const TcpConnectionPtr&,
                       const string& message,
                       Timestamp)
  {
  /*引用计数加1，mutex保护的临时区大大缩短*/
    ConnectionListPtr connections = getConnectionList();;//栈上变量
  /*可能大家会有疑问，不受mutex保护，写者更改了连接列表怎么办�*
    实际上，写者是在另一个副本上修改，所以无需担心*/
    for (ConnectionList::iterator it = connections->begin();
        it != connections->end();
        ++it)
    {
    /*这里也是无法减少第一个和第二个连接发送所需的时间,
    因为他们都是在同步发送的，就是所要等到转发完一条消息到
    一个connection后,然后才能转发下一个连接connection.
    实质就是调用这个函数的IO负责转发*/
      codec_.send(get_pointer(*it), message);
    }
        /*这个断言不一定成立
        assert(!connections.uniquer())。
        这是由于Onconnection()---->connections_.reset(new ConnectionList(*connections_));*/
        /*当connection这个栈上的变量销毁的时候，引用计数减1*/
  }
 
  ConnectionListPtr getConnectionList()
  {
  /*保护区域变小了<>*/
    MutexLockGuard lock(mutex_);
    return connections_;
  }
 
  EventLoop* loop_;
  TcpServer server_; /*tcpserver服务器*/
  LengthHeaderCodec codec_;
  MutexLock mutex_;
  ConnectionListPtr connections_;
};
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid();
  if (argc > 1)
  {
    EventLoop loop;
    uint16_t port = static_cast<uint16_t>(atoi(argv[1]));
    InetAddress serverAddr(port);
    ChatServer server(&loop, serverAddr);
    if (argc > 2)
    {
    /*IO线程个数*/
      server.setThreadNum(atoi(argv[2]));
    }
    server.start();
    loop.loop();
  }
  else
  {
    printf("Usage: %s port [thread_num]\n", argv[0]);
  }
}
```

### examples/asio/chat/server_threaded_highperformance.cc，采用thread local变量实现多线程高效转发 

``` c++
#include "codec.h"
 
#include <muduo/base/Logging.h>
#include <muduo/base/Mutex.h>
#include <muduo/base/ThreadLocalSingleton.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/TcpServer.h>
 
#include <boost/bind.hpp>
#include <boost/shared_ptr.hpp>
 
#include <set>
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
/*这个主要是针对第二个进行改正的，*/
class ChatServer : boost::noncopyable
{
 public:
  ChatServer(EventLoop* loop,
             const InetAddress& listenAddr)
  : loop_(loop),
    server_(loop, listenAddr, "ChatServer"),
    codec_(boost::bind(&ChatServer::onStringMessage, this, _1, _2, _3))
  {
    server_.setConnectionCallback(
        boost::bind(&ChatServer::onConnection, this, _1));
    server_.setMessageCallback(
        boost::bind(&LengthHeaderCodec::onMessage, &codec_, _1, _2, _3));
  }
 
  void setThreadNum(int numThreads)
  {
  /*设置sub IO线程池的大小*/
    server_.setThreadNum(numThreads);
  }
 
  void start()
  {/*设置线程的初始化函数*/
    server_.setThreadInitCallback(boost::bind(&ChatServer::threadInit, this, _1));
    server_.start();
  }
 
 private:
  void onConnection(const TcpConnectionPtr& conn)
  {
    LOG_INFO << conn->localAddress().toIpPort() << " -> "
             << conn->peerAddress().toIpPort() << " is "
             << (conn->connected() ? "UP" : "DOWN");
 
    if (conn->connected())
    {
      connections_.instance().insert(conn);
    }
    else
    {
      connections_.instance().erase(conn);
    }
    cout<<"connection adress :"<<&connections_<<"\t"<<"connection size :"<<connections_.size() ;
  }
 
  void onStringMessage(const TcpConnectionPtr&,
                       const string& message,
                       Timestamp)
  {
  /*把消息"转发"作为IO线程的任务来处理*/
    EventLoop::Functor f = boost::bind(&ChatServer::distributeMessage, this, message);
    LOG_DEBUG;
    /*转发消息给所有客户端，高效率(多线程来转发),转发到不同的IO线程,
 
    */
    MutexLockGuard lock(mutex_);
    /*for 循环和f达到异步进行*/
    for (std::set<EventLoop*>::iterator it = loops_.begin();
        it != loops_.end();
        ++it)
    {/*
    1.让对应的IO线程来执行distributeMessage
    2.distributeMessage放到IO线程队列中执行，因此，这里的mutex_锁竞争大大减小
    3.distributeMesssge 不受mutex_保护
            */
      (*it)->queueInLoop(f);
    }
    LOG_DEBUG;
  }
 
  typedef std::set<TcpConnectionPtr> ConnectionList;
 
  void distributeMessage(const string& message)
  {
    LOG_DEBUG << "begin";
    // connectionList_是线程局部变量
    for (ConnectionList::iterator it = connections_.instance().begin();
        it != connections_.instance().end();
        ++it)
    {
      codec_.send(get_pointer(*it), message);
    }
    LOG_DEBUG << "end";
  }
/*IO线程执行前时的前回调函数*/
  void threadInit(EventLoop* loop)
  {
    assert(connections_.pointer() == NULL);
    /*实例化一个对象*/
    connections_.instance();
    assert(connections_.pointer() != NULL);
    MutexLockGuard lock(mutex_);
    loops_.insert(loop);
  }
 
  EventLoop* loop_; //loop_传递给server_
  TcpServer server_;
  LengthHeaderCodec codec_;
  /*线程局部单例变量，每个线程都有一个connections_(连接列表)实例*/
  ThreadLocalSingleton<ConnectionList> connections_;
 
  MutexLock mutex_;
  std::set<EventLoop*> loops_;        //eventLoop列表
};
 
int main(int argc, char* argv[])
{
  LOG_INFO << "pid = " << getpid();
  if (argc > 1)
  {
    EventLoop loop;//acceptor loop
    uint16_t port = static_cast<uint16_t>(atoi(argv[1]));
    InetAddress serverAddr(port);
    ChatServer server(&loop, serverAddr);
    if (argc > 2)
    {
      server.setThreadNum(atoi(argv[2])); //多个subIO线程
    }
    server.start();
    loop.loop();
  }
  else
  {
    printf("Usage: %s port [thread_num]\n", argv[0]);
  }
}
```

## [47] 限制服务器最大并发连接数

限制服务器最大并发连接数（`MuduoManual.pdf P108`）
用`Timing wheel`踢掉空闲连接（`MuduoManual.pdf P122`）

### Timing wheel 

echo.h

``` c++
#ifndef MUDUO_EXAMPLES_IDLECONNECTION_ECHO_H
#define MUDUO_EXAMPLES_IDLECONNECTION_ECHO_H
 
#include <muduo/net/TcpServer.h>
//#include <muduo/base/Types.h>
 
#include <boost/circular_buffer.hpp>
#include <boost/unordered_set.hpp>
#include <boost/version.hpp>
 
#if BOOST_VERSION < 104700
namespace boost
{
template <typename T>
inline size_t hash_value(const boost::shared_ptr<T>& x)
{
  return boost::hash_value(x.get());
}
}
#endif
 
// RFC 862
class EchoServer
{
 public:
  EchoServer(muduo::net::EventLoop* loop,
             const muduo::net::InetAddress& listenAddr,
             int idleSeconds);
 
  void start();
 
 private:
  void onConnection(const muduo::net::TcpConnectionPtr& conn);
 
  void onMessage(const muduo::net::TcpConnectionPtr& conn,
                 muduo::net::Buffer* buf,
                 muduo::Timestamp time);
 
  void onTimer();
 
  void dumpConnectionBuckets() const;
 
  typedef boost::weak_ptr<muduo::net::TcpConnection> WeakTcpConnectionPtr;
 
  struct Entry : public muduo::copyable
  {
    explicit Entry(const WeakTcpConnectionPtr& weakConn)
      : weakConn_(weakConn) //这是一个弱指针，所以创建一个对象时，引用计数不会加一
    {
    }
 
    ~Entry()
    {/*当引用计数为0时，会调用虚构函数；
将弱指针提升为强指针，然后关闭连接
    */
      muduo::net::TcpConnectionPtr conn = weakConn_.lock();
      if (conn)
      {
        conn->shutdown();
      }
    }
 
    WeakTcpConnectionPtr weakConn_;
  };
  typedef boost::shared_ptr<Entry> EntryPtr; //共享型Entry指针
  typedef boost::weak_ptr<Entry> WeakEntryPtr;//弱指针Entry型
  typedef boost::unordered_set<EntryPtr> Bucket;//共享型Entry集合
  typedef boost::circular_buffer<Bucket> WeakConnectionList;
 
  muduo::net::EventLoop* loop_;
  muduo::net::TcpServer server_;
  WeakConnectionList connectionBuckets_;
};
 
#endif  // MUDUO_EXAMPLES_IDLECONNECTION_ECHO_H
```

echo.cc

``` c++
#include "echo.h"
 
#include <muduo/base/Logging.h>
#include <muduo/net/EventLoop.h>
 
#include <boost/bind.hpp>
 
#include <assert.h>
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
 
EchoServer::EchoServer(EventLoop* loop,
                       const InetAddress& listenAddr,
                       int idleSeconds)
  : loop_(loop),
    server_(loop, listenAddr, "EchoServer"),
    connectionBuckets_(idleSeconds)
{
  server_.setConnectionCallback(
      boost::bind(&EchoServer::onConnection, this, _1));
  server_.setMessageCallback(
      boost::bind(&EchoServer::onMessage, this, _1, _2, _3));
  loop->runEvery(1.0, boost::bind(&EchoServer::onTimer, this));
  connectionBuckets_.resize(idleSeconds);
  dumpConnectionBuckets();
}
 
void EchoServer::start()
{
  server_.start();
}
/*连接到来时的回调函数*/
void EchoServer::onConnection(const TcpConnectionPtr& conn)
{
  LOG_INFO << "EchoServer - " << conn->peerAddress().toIpPort() << " -> "
           << conn->localAddress().toIpPort() << " is "
           << (conn->connected() ? "UP" : "DOWN");
 
  if (conn->connected())
  {
  /*将连接和entryptr绑定
  引用计数为1，这是一个临时对象*/
    EntryPtr entry(new Entry(conn));
  /*插入到队尾，这时候引用计数位2*/
    connectionBuckets_.back().insert(entry);
  /*打出桶的连接数*/
    dumpConnectionBuckets();
    WeakEntryPtr weakEntry(entry);
    /*设置一下上下文，不使用强引用，防止引用计数加1
    这也是为了在OnMessage()函数调用时可以使用
    */
    conn->setContext(weakEntry);
  }//临时对象无效，引用计数减1
 
  else
  {
    assert(!conn->getContext().empty());
    /*输出一下引用计数*/
    WeakEntryPtr weakEntry(boost::any_cast<WeakEntryPtr>(conn->getContext()));
    LOG_DEBUG << "Entry use_count = " << weakEntry.use_count();
  }
}
 
/*消息到达时的回调函数*/
void EchoServer::onMessage(const TcpConnectionPtr& conn,
                           Buffer* buf,
                           Timestamp time)
{
  string msg(buf->retrieveAllAsString());
  LOG_INFO << conn->name() << " echo " << msg.size()
           << " bytes at " << time.toString();
  conn->send(msg);
 
  assert(!conn->getContext().empty());
  WeakEntryPtr weakEntry(boost::any_cast<WeakEntryPtr>(conn->getContext()));
  EntryPtr entry(weakEntry.lock());//+1
  if (entry)
  {
  /*在tail后面插入一个entry*/
    connectionBuckets_.back().insert(entry);//+1
    dumpConnectionBuckets();
  }//-1
}
 
/*时钟达到*/
void EchoServer::onTimer()
{
/*清空该位置上的集合，集合里面的指针引用计数都减1*/
  connectionBuckets_.push_back(Bucket());
  dumpConnectionBuckets();
}
/*打出桶的连接数*/
 
void EchoServer::dumpConnectionBuckets() const
{
  LOG_INFO << "size = " << connectionBuckets_.size();
  int idx = 0;
  /*弱引用*/
  for (WeakConnectionList::const_iterator bucketI = connectionBuckets_.begin();
      bucketI != connectionBuckets_.end();
      ++bucketI, ++idx)
  {
    const Bucket& bucket = *bucketI;
    printf("[%d] len = %zd : ", idx, bucket.size());
    for (Bucket::const_iterator it = bucket.begin();
        it != bucket.end();
        ++it)
    {
      bool connectionDead = (*it)->weakConn_.expired();
      printf("%p(%ld)%s, ", get_pointer(*it), it->use_count(),
          connectionDead ? " DEAD" : "");
    }
    puts("");
  }
}
```

sortedlist.h

``` c++
#include <muduo/base/Logging.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/TcpServer.h>
#include <boost/bind.hpp>
#include <list>
#include <stdio.h>
 
using namespace muduo;
using namespace muduo::net;
 
// RFC 862
class EchoServer
{
 public:
  EchoServer(EventLoop* loop,
             const InetAddress& listenAddr,
             int idleSeconds);
 
  void start()
  {
    server_.start();
  }
 
 private:
  void onConnection(const TcpConnectionPtr& conn);
 
  void onMessage(const TcpConnectionPtr& conn,
                 Buffer* buf,
                 Timestamp time);
 
  void onTimer();
 
  void dumpConnectionList() const;
 
  typedef boost::weak_ptr<TcpConnection> WeakTcpConnectionPtr;//弱连接指针
  typedef std::list<WeakTcpConnectionPtr> WeakConnectionList;//连接列表(元素是指针)
 
  struct Node : public muduo::copyable
  {
    Timestamp lastReceiveTime;  //该连接最后一次活跃时刻
    WeakConnectionList::iterator position; //该连接在连接列表中的位置
  };
 
  EventLoop* loop_;
  TcpServer server_;
  int idleSeconds_;
  WeakConnectionList connectionList_;//连接列表
};
 
EchoServer::EchoServer(EventLoop* loop,
                       const InetAddress& listenAddr,
                       int idleSeconds)
  : loop_(loop),
    server_(loop, listenAddr, "EchoServer"),
    idleSeconds_(idleSeconds)
{
  server_.setConnectionCallback(
      boost::bind(&EchoServer::onConnection, this, _1));
  server_.setMessageCallback(
      boost::bind(&EchoServer::onMessage, this, _1, _2, _3));
  loop->runEvery(1.0, boost::bind(&EchoServer::onTimer, this));
  dumpConnectionList();
}
 
void EchoServer::onConnection(const TcpConnectionPtr& conn)
{
  LOG_INFO << "EchoServer - " << conn->peerAddress().toIpPort() << " -> "
           << conn->localAddress().toIpPort() << " is "
           << (conn->connected() ? "UP" : "DOWN");
 
  if (conn->connected())
  {
    Node node;
    node.lastReceiveTime = Timestamp::now();
    connectionList_.push_back(conn);
    node.position = --connectionList_.end();
    conn->setContext(node);
  }
  else
  {
    assert(!conn->getContext().empty());
    const Node& node = boost::any_cast<const Node&>(conn->getContext());
    connectionList_.erase(node.position);
  }
  dumpConnectionList();
}
 
void EchoServer::onMessage(const TcpConnectionPtr& conn,
                           Buffer* buf,
                           Timestamp time)
{
  string msg(buf->retrieveAllAsString());
  LOG_INFO << conn->name() << " echo " << msg.size()
           << " bytes at " << time.toString();
  conn->send(msg);
 
  assert(!conn->getContext().empty());
  Node* node = boost::any_cast<Node>(conn->getMutableContext());
  node->lastReceiveTime = time;
  // move node inside list with list::splice()
  connectionList_.erase(node->position);
  connectionList_.push_back(conn);
  node->position = --connectionList_.end();
 
  dumpConnectionList();
}
/*时钟回调函数*/
void EchoServer::onTimer()
{
  dumpConnectionList();
  Timestamp now = Timestamp::now();
  for (WeakConnectionList::iterator it = connectionList_.begin();
      it != connectionList_.end();)
  {
    TcpConnectionPtr conn = it->lock();
    if (conn)
    {
      Node* n = boost::any_cast<Node>(conn->getMutableContext());
      double age = timeDifference(now, n->lastReceiveTime);
      if (age > idleSeconds_)
      {/*剔除超时的连接*/
        conn->shutdown();
      }
      else if (age < 0)
      {
        LOG_WARN << "Time jump";
        n->lastReceiveTime = now;
      }
      else
      {
        break;
      }
      ++it;
    }
    else
    {
      LOG_WARN << "Expired";
      it = connectionList_.erase(it);
    }
  }
}
 
void EchoServer::dumpConnectionList() const
{
  LOG_INFO << "size = " << connectionList_.size();
 
  for (WeakConnectionList::const_iterator it = connectionList_.begin();
      it != connectionList_.end(); ++it)
  {
    TcpConnectionPtr conn = it->lock();
    if (conn)
    {
      printf("conn %p\n", get_pointer(conn));
      const Node& n = boost::any_cast<const Node&>(conn->getContext());
      printf("    time %s\n", n.lastReceiveTime.toString().c_str());
    }
    else
    {
      printf("expired\n");
    }
  }
}
 
int main(int argc, char* argv[])
{
  EventLoop loop;
  InetAddress listenAddr(2007);
  int idleSeconds = 10;
  if (argc > 1)
  {
    idleSeconds = atoi(argv[1]);
  }
  LOG_INFO << "pid = " << getpid() << ", idle seconds = " << idleSeconds;
  EchoServer server(&loop, listenAddr, idleSeconds);
  server.start();
  loop.loop();
}
```
``` c++
#include "echo.h"
#include <stdio.h>
 
#include <muduo/base/Logging.h>
#include <muduo/net/EventLoop.h>
 
using namespace muduo;
using namespace muduo::net;
 
void testHash()
{
  boost::hash<boost::shared_ptr<int> > h;
  boost::shared_ptr<int> x1(new int(10));
  boost::shared_ptr<int> x2(new int(10));
  h(x1);
  assert(h(x1) != h(x2));
  x1 = x2;
  assert(h(x1) == h(x2));
  x1.reset();
  assert(h(x1) != h(x2));
  x2.reset();
  assert(h(x1) == h(x2));
}
 
int main(int argc, char* argv[])
{
  testHash();
  EventLoop loop;
  InetAddress listenAddr(2007);
  int idleSeconds = 10;
  if (argc > 1)
  {
    idleSeconds = atoi(argv[1]);
  }
  LOG_INFO << "pid = " << getpid() << ", idle seconds = " << idleSeconds;
  EchoServer server(&loop, listenAddr, idleSeconds);
  server.start();
  loop.loop();
}
```

