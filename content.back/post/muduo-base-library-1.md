---
title: muduo_base_library_1
date: 2013-06-10 01:09:28
tags:
- muduo 
- muduo base library
categories:
- muduo 
- network program
description: muduo 是一个linux 网络库，使用poll和epoll模式。 这些内容是本人在大学的时候，通过  << c++教程网 >>  所学的知识笔记. 已经过去好几年了，版本也是比较老的了，属于入门级别的教程； 如有错误，纯属正常!!!
---

# Muduo Base Library 

### 面向对象编程

面向对象也就是把`对象`作为`接口`暴露出去，一般内部是一个`接口`或者`抽象类`。


*source code*  


- Thread.h

``` c++

#ifndef _THREAD_H_ 
#define _THREAD_H_
#include <pthread.h>
class Thread{
public :
 
    Thread();
    virtual ~Thread();
    void Start();
    void Join();
 
private :
    // 纯虚函数 ²»ÐҪ 
    void Run() =0 ;
    pthread_t threadId_ ; 
 
};#ifndef _THREAD_H_ 
#define _THREAD_H_
#include <pthread.h>
class Thread{
public :
 
    Thread();
    virtual ~Thread();
    void Start();
    void Join();
 
private :
    // 纯虚函数 ²»ÐҪ 
    void Run() =0 ;
    pthread_t threadId_ ; 
 
};

```
- Thread.cpp

```c++

#include "Thread.h"
#include <iostream>
using namespace std;
 
 
Thread::Thread() : autoDelete_(false)
{
    cout<<"Thread ..."<<endl;
}
 
Thread::~Thread()
{
    cout<<"~Thread ..."<<endl;
}
 
void Thread::Start()
{
    pthread_create(&threadId_, NULL, ThreadRoutine, this);
}
 
void Thread::Join()
{
    pthread_join(threadId_, NULL);
}
 
void* Thread::ThreadRoutine(void* arg)
{
    Thread* thread = static_cast<Thread*>(arg);
    thread->Run();
    if (thread->autoDelete_)
        delete thread;
    return NULL;
}
 
void Thread::SetAutoDelete(bool autoDelete)
{
    autoDelete_ = autoDelete;
}

```

- Thread test 

``` c++

#include "Thread.h"
#include <unistd.h>
#include <iostream>
using namespace std;
 
class TestThread : public Thread
{
public:
    TestThread(int count) : count_(count)
    {
        cout<<"TestThread ..."<<endl;
    }
 
    ~TestThread()
    {
        cout<<"~TestThread ..."<<endl;
    }
 
private:
    void Run()
    {
        while (count_--)
        {
            cout<<"this is a test ..."<<endl;
            sleep(1);
        }
    }
 
    int count_;
};
 
int main(void)
{
    /*
    TestThread t(5);
    t.Start();
 
    t.Join();
    */
 
    TestThread* t2 = new TestThread(5);
    t2->SetAutoDelete(true);
    t2->Start();
    t2->Join();
 
    for (; ; )
        pause();
 
    return 0;
}


```

### 基于对象编程

基于对象编程，提供出来的就不是`接口`了， 而是一个具体的类，虽然他和面向对象想一个都是起到回调的作用，但是他的回调不是通过继承来实现相应的虚函数,可以独立的函数，或者成员函数。看下面的例子就知道了。

*source code*

- Thread.h

``` c++

#ifndef _THREAD_
#define _THREAD_
#include <pthread.h>
#include <boost/function.hpp>
class Thread {
public :
    typedef boost::function<void ()> ThreadFunc ;
    explicit Thread( const ThreadFunc &func) ;
    ~Thread();
    void Start () ;
    void Join();
    void SetAutoDelete(bool autoDelete);
private :
    static void* ThreadRoutine( void *arg) ;
    void Run();
    ThreadFunc func ;
    pthread_t threadId ;
    bool autoDelete_;
} ;
#endif // _THREAD_H_

```

- Thread.cpp

``` c++

#include "Thread.h"
#include <boost/bind.hpp>
#include <iostream>
using namespace std ;
Thread::Thread( const ThreadFunc &func_ ) : func(func_),autoDelete_(false){
 
}
Thread::~Thread()
{
    cout<<__FUNCTION__<<endl;
}
 
 
void Thread::Start()
{
    pthread_create(&threadId, NULL, ThreadRoutine, this);
}
 
void Thread::Join(){
 
    pthread_join(threadId,NULL);
}
void * Thread::ThreadRoutine( void *arg){
    Thread *thread = static_cast<Thread*>(arg) ;
    thread->Run();
    if ( thread->autoDelete_)
        delete(thread);
    cout<<"After delete"<<endl;
 
 
 
}
void Thread::Run(){
    func();
}
void Thread::SetAutoDelete(bool autoDelete)  {
    this->autoDelete_ = autoDelete;
 

```

- Thread_test.cpp


``` c++

#include <unistd.h>
#include "Thread.h"
#include <iostream>
#include <boost/bind.hpp>
using namespace std ;
 
class NumberFunction{
public :
    NumberFunction( int count):coun_t(count){}
        void NFunction() {
        while(coun_t-- && coun_t>0){
            cout<<__FUNCTION__<<endl;
            sleep(1);
        }
    }
private :
    int coun_t ;
};
 
void threadFunc (){
 
    int i = 9 ;
    while(i--){
        cout<<"threadFunc...!!"<<endl ;
        sleep(1);
    }
 
    cout<<endl ;
}
 
void threadFunc2(int i ){
    while(i--&&i>=0){
        cout<<"Threadfunc2 ...."<<endl;
        sleep(1);
    }
 
}
int main (){
 
    Thread *t = new Thread(threadFunc);
    Thread *t2 = new Thread(boost::bind( threadFunc2 , 10)) ;
    NumberFunction foo(10) ;
    Thread *t3 = new Thread( boost::bind(&NumberFunction::NFunction,&foo)) ;
    t->SetAutoDelete(true);
    t->Start();
 
    t2->SetAutoDelete(true) ;
    t2->Start();
 
    t3->SetAutoDelete(true) ;
    t3->Start();
 
    t2->Join();
    t->Join();
    t3->Join();
    int i = 9 ;
    while (i--){
        cout<<__FUNCTION__<<endl;
    }
 
 
    return 0 ;
}

```

我们看一下下面这张图

<center>![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/muduo_base_library_1.png)</center> 

``` c++

class EchoServer
{
    public :
        EchoServer(){
            server.SetConnectionCallback (boost::bind( onConnection()) ;
            server.SetConnectionCallback (boost::bind( onMessage() ) ;
            server.SetConnectionCallback (boost::bind( onClose() ) ;
            TcpServer server ;
        }
}

```

*面向对象风格*: 

> 用一个EchoServer 继承TcpServer(抽象类)，然后EchoServer实现三个接口onConnection、onMessage、onClose。

*基于对象风格*:

> 用一个EchoServer包含一个TcpServer(具体类)对象，在构造函数中使用boost::bind来注册 三个成员函数 onConnection、onMessage、onClose。


### poll

`signal (SIGPIPE ,SIG_IGN)`: 　

Linux网络编程 第12讲 tcp 11中状态

- `ＳＩＧＰＩＰＥ`产生的原因：        

如果客户端关闭套接字`close` 而服务器调用一次`RST segment`(TCP传输层)如果服务器再次调用了`write` ，这个时候就会产生`SIGPIPE`信号,

- `TIMEOUT`会对大并发服务器产生很大的影响

由于服务器主动关闭连接，服务器就会进入`TIME_WAIT`,所以我们要使`client`先关闭，服务器被动关闭；

- 踢掉不活跃的客户端

如果客户端不活跃了，一些客户端不断开连接，这样子就会占用服务器的连接资源，服务器端也有要个机制来踢掉不活跃的连接，即使服务器又会进入time-wait状态。

- 客户端关闭在`poll`中属于`pollin`事件，并且可读的数据为`0`； 对于监听套接字来说，有连接请求也属于`pollin`事件。
 

所以我们使用了`nonblocking sokcet + I/O`复用

``` c++ 

#include <unistd.h>
#include <sys/types.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <sys/wait.h>
#include <poll.h>
 
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
 
#include <vector>
#include <iostream>
 
#define ERR_EXIT(m) \
        do \
        { \
                perror(m); \
                exit(EXIT_FAILURE); \
        } while(0)
 
typedef std::vector<struct pollfd> PollFdList;
 
int main(void)
{
    // 防止进程由于客户端的关闭产生SIG_PIPE信号而是服务器进程退出
    signal(SIGPIPE, SIG_IGN);
    //SIGCHLD 子进程状态改变后会产生此信号，父进程需要调用一个wait函数以确定发生了什么。
    //如果进程特定设置该信号的配置为SIG_IGN，则调用进程的子进程不产生僵死进程。
    //其实要想防止进程僵死，我们应该在他的父进程里面调用wait，以获取子进程的状态就不会让子进程变成僵死进程了
    signal(SIGCHLD, SIG_IGN);
 
    //int idlefd = open("/dev/null", O_RDONLY | O_CLOEXEC);
    int listenfd;
 
    //if ((listenfd = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
    //SOCK_NONBLOCK:非阻塞 ，2.6.2**才有
    //SOCK_CLOEXEC :当进程调用vfork或者fork产生进程时，继承下来的描述是关闭的状态
    //
    if ((listenfd = socket(PF_INET, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, IPPROTO_TCP)) < 0)
        ERR_EXIT("socket");
 
    struct sockaddr_in servaddr;
    memset(&servaddr, 0, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(5188);
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
 
    int on = 1;
    //SOL_SOCKET:在套接字级别上进行设置
    // SO_REUSEADDR，打开或关闭地址复用功能。
    if (setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0)
        ERR_EXIT("setsockopt");
 
    if (bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
        ERR_EXIT("bind");
    if (listen(listenfd, SOMAXCONN) < 0)
        ERR_EXIT("listen");
 
    struct pollfd pfd;
    pfd.fd = listenfd;
    //注册事件
    pfd.events = POLLIN;
 
    PollFdList pollfds;
    pollfds.push_back(pfd);
    //a positive number is returned
    int nready;
 
    struct sockaddr_in peeraddr;
    socklen_t peerlen;
    int connfd;
 
    while (1)
    {
        //可用POLLIN事件数
        //  -1 ：无限等待，直到发送POLLIN事件
        nready = poll(&*pollfds.begin(), pollfds.size(), -1);
        if (nready == -1)
        {
            if (errno == EINTR)
                continue;
 
            ERR_EXIT("poll");
        }
        if (nready == 0)    // nothing happended
            continue;
 
        //如果POLLIN事件里面包含监听socket，那么监听socket将会
        if (pollfds[0].revents & POLLIN)
        {
            peerlen = sizeof(peeraddr);
            //SOCK_NONBLOCK : 非阻塞模式
            connfd = accept4(listenfd, (struct sockaddr*)&peeraddr,
                        &peerlen, SOCK_NONBLOCK | SOCK_CLOEXEC);
 
            if (connfd == -1)
                ERR_EXIT("accept4");
 
/*
            if (connfd == -1)
            {
                if (errno == EMFILE)
                {
                    close(idlefd);
                    idlefd = accept(listenfd, NULL, NULL);
                    close(idlefd);
                    idlefd = open("/dev/null", O_RDONLY | O_CLOEXEC);
                    continue;
                }
                else
                    ERR_EXIT("accept4");
            }
*/
 
            pfd.fd = connfd;
            pfd.events = POLLIN;
            pfd.revents = 0;
            pollfds.push_back(pfd);
            --nready;
 
            // 连接成功
            std::cout<<"ip="<<inet_ntoa(peeraddr.sin_addr)<<
                " port="<<ntohs(peeraddr.sin_port)<<std::endl;
            if (nready == 0)
                continue;
        }
 
        //std::cout<<pollfds.size()<<std::endl;
        //std::cout<<nready<<std::endl;、
        //过滤掉第一个socketfd，也就是监听sdocket
        for (PollFdList::iterator it=pollfds.begin()+1;
            it != pollfds.end() && nready >0; ++it)
        {
            //如果有POLLIN事件
                if (it->revents & POLLIN)
                {
                    --nready;
                    connfd = it->fd;
                    char buf[1024] = {0};
                    int ret = read(connfd, buf, 1024);
                    if (ret == -1)
                        ERR_EXIT("read");
                    //如果等于0 ，就是说客户端已关闭，既然是POLLIN事件，那么就表示有数据可读，但现在为0（），也就是所读到了文件的EOF
                    if (ret == 0)
                    {
                        std::cout<<"client close"<<std::endl;
                        it = pollfds.erase(it);
                        --it;
 
                        close(connfd);
                        continue;
                    }
 
                    std::cout<<buf;
                    write(connfd, buf, strlen(buf));
 
                }
        }
    }
 
    return 0;
}

```

### poll2

```

application buffer          kernel buffer

|------|                |--------|
| read | <--------------| buffer |
|------|                |--------|


|------|                |--------|
|write | <--------------| buffer |
|------|                |--------|

```

数据包　粘包　一个包　两次`read`
- Read 
可能并没有把`confd`对应的缓冲区的数据读完，那么`connfg`仍然是活跃的，我们应该将读到的数据保存在`connfd`的应用层的缓冲区（每一次都进行追加）。如何解析协议，我们让应用层的解析协议自己来解析
- 忙等待 
 假设客户端关注了`socket`的`POLLOUT`事件，而此时内核缓冲区有空闲，但是应用层却没数据可写，那么内核将会处于忙等待状态(`busy waitting loop`)，一直发送`POLLOUT`事件。

解决的方法是：我们要看应用层的缓冲区，如果应用层的缓冲区有数据发，那么我们应该关注`POLLOUT`事件，要不然就取消`POLLOUT`事件的关注。

对于客户端在读数据时，我们也应该采用相应的方法，如果应用层的空间空闲时，我们就关注`POLLIN`事件，要不然就取消`POLLIN`事件。


### epoll 1

``` c++ 
/*
 poll 在已连接的套接字中遍历
 epoll_wait 返回的都是活跃的套接字，所以减少了很多无效的套接字
 
 Poll模型 ：
 每次调用 poll函数的时候。都需要把监听套接字与已连接套接字所感兴趣的事件数组 拷贝到内核。
 
 
 LT模式 ：
 Write  EPOLLOUT 事件
    高电平 writebuf内核有空闲空间，我们就说他处于高电平状态，也就是一直处于活跃状态。此时可能会产生busy waitting loop
    低电平 当writebuf内核没有空闲空间，我们就说他处于低电平状态，没有激活。
Read EPOLLIN 事件
    xxx
 
 
 
 ET模型 边缘触发
 在该模式下，一开始我们就关注EPOLLIN事件和EPOLLOUT事件 ， 此时writbuf一直处于高电平，不会触发EPOLLOUT事件.
        而EPOLLIN可能会产生，因为开始Readbuf是空的，如果在 epoll_wait前，readbuf有数据了，那么就有unreadablr--->readable,
        也就是产生了EPOLLLIN事件，这也是为什么监听socket 能够接收到外来请求的原因。
        关注EPOLLIN 和 EPOLLOUT事件后，我们也没必要取消他们的关注，只有到断开他们的socketfd时 ， 我们才需要取消。
 
需要注意的是，在读写时，如果是读，一定要读到EAGAIN，写也是要写到EAGAIN
 socket从unreadable变为readable或从unwritable变为writable
 有些人说从readable变为unreadable或者writable变为unwritable时也会触发事件。
 我个人觉得第一种合理一点。
 
 
*/
 
#include <unistd.h>
#include <sys/types.h>
#include <fcntl.h>
#include <sys/socket.h>
#include <netnet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/epoll.h>
#include <stdio.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
 
#include <vector.h>
#include <algorithm>
#include <iostream>
 
typedef std::vector<struct epoll_event> EventList ;
 
#define ERR_EXIT(m) \
    do\
    {\
        perror(m);\
        exit(EXIT_FAILURE);\
    }while(0);
 
int main ( void ) {
    //防止进程由于客户端的关闭而使服务器进程退出
    signal(SIGPIPE ,SIG_IGN) ;  
    //防止僵死进程的发生
    signal(SIGCHLD,SIG_IGN);
 
 
    //备胎描述符
    int idlefd = open("/dev/null",O_RDONLY | O_CLOEXEC) ;
    //监听描述符
    int listenfd ;
 
    if((listenfd =socket(PF_INET,SOCK_STRAM|SOCK_NONBLOCK|SOCK_CLOSEXEC,IPPROTO_TCP))<0){
 
        ERR_EXIT("socket");
    }
 
    struct sockaddr_in servaddr ;
    memset(&servaddr,0,sizeof(servaddr)) ;
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(5188);
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY) ;
 
    int on =1 ; 
    //socketfd 重用
    if(setsocketopt(listenfd,SOL_SOCKET,SO_REUSERADDR,&on,sizeof(n))<0)
 
    {
        ERR_EXIT("setsocketopt");
    }
 
    if( bind(listenfd,(struct sockaddr*)&servaddr,sizeof(servaddr)) <0)
        ERR_EXIT("bind");
 
    if(listen(listen,SOMAXCONN) <0)
        ERR_EXIT("listen");
 
    std::vector<int> clients;
    int epollfd; 
    // 函数返回一个epoll专用的描述符epfd，epfd引用了一个新的epoll机制例程（instance.）。
    epollfd = epoll_create1(EPOLL_CLOEXEC);
 
    //事件结构体
    struct epoll_event event ;
    event.data.fd = listenfd ; //关注listenfd
    event.events = EPOLLIN ; // /* | EPOLLET*/; 默认是LT模型
 
    // 把lisenfd 添加到epollfd 中进行管理
    //如果是poll的话，就不用这样了，poll直接使用一个数组就行了
    /*函数声明：int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event) 
该函数用于控制某个epoll文件描述符上的事件，可以注册事件，修改事件，删除事件。 
参数： 
epfd：由 epoll_create 生成的epoll专用的文件描述符； 
op：要进行的操作例如注册事件，可能的取值EPOLL_CTL_ADD 注册、EPOLL_CTL_MOD 修 改、EPOLL_CTL_DEL 删除 
 
fd：关联的文件描述符； 
event：指向epoll_event的指针； 
如果调用成功返回0,不成功返回-1 
 
71 - 78 就交给epollfd 内核帮我们做了，并且只有epoll_ctl才会拷贝到内核，一次拷贝就行了，以后就不用拷贝了，
如果我们使用poll 那么每次循环都要拷贝到内核里面去，大大降低了效率
        */
    epoll_ctl(epollfd , EPOLL_CTL_ADD,listenfd,&event);
 
    //事件列表，初始为16 个
    EventList events(16) ;
    struct sockaddr_in peeraddr ;
    socklen_t peerlen ;
    int connfd ;
 
    int nready ; 
    while(1){
 
/*函数声明:int epoll_wait(int epfd,struct epoll_event * events,int maxevents,int timeout)
该函数用于轮询I/O事件的发生；
参数：
epfd:由epoll_create 生成的epoll专用的文件描述符；
epoll_event:用于回传代处理事件的数组，要处理的事件都会存储在events里面了
maxevents:每次能处理的事件数；
timeout:等待I/O事件发生的超时值(单位我也不太清楚)；-1相当于阻塞，0相当于非阻塞。一般用-1即可
返回发生事件数。
         -1 表示等待到至少一个事件发生*/
        nready = epoll_wait(epollfd,&*events.begin(),static_cast<int>(events.size()), -1 ); 
        if( nready == -1){
 
            if(errno == EINTR)
                continue ;
            ERR_EXIT("epoll_wait") ;
        }
 
        if(nready ==0 ) //nothing to happened 
            continue ;
        // 
        if((size_t)nready == events.size())
            events.resize(events.size()*2) ;
 
        for (int i = 0; i < nready; ++i)
        {
            //如果是监听socket，则accetp
            if (events[i].data.fd == listenfd)
            {
                peerlen = sizeof( peeraddr ) ;
                // 这里的处理还不够好，因为accept4每一次只能到tcp就绪队列里面拿出一个就绪socketfd ， 有可能这个队列不止一个，
                //在并发的时候这是必然的， 所以最后是while掉accept4，把里面的就绪socketfd全部拿出来，而不时只拿一个socketfd。
                //下面的代码可根据需求进行改进
                connfd =::accept4(listenfd,(struct sockaddr *)&peeraddr,&peerlen,SOCK_NONBLOCK|SOCK_CLOSEXEC) ;
 
                if( connfd == -1 )
                {
 
                    if(errno == EMFILE){
                        close(idlefd) ;
                        idlefd = accept(listenfd,NULL,NULL) ;
                        close(idlefd) ;
                        idlefd = open("/dev/null",O_RDONLY|O_CLOEXEC) ;
                    }
 
                }
                else
                    ERR_EXIT("accept4") ;
                std::cout<<"ip="<<inet_ntoa(peeraddr.sin_addr)<<" port="<<ntohs(peeraddr.sin_port)<<std::endl;
                clients.push_back(connfd) ;
 
                event.data.fd =connfd ;
                event.events = EPOLLIN/*| EPOLLET*/ ;
                epoll_ctl(epollfd,EPOLL_CTL_ADD ,connfd ,&event) ;
            }else if (events[i].events & EPOLLIN)
            {
                //下面的这些都要改进
                connfd = events[i].data.fd;
                if (connfd < 0)
                    continue;
 
                char buf[1024] = {0};
                int ret = read(connfd, buf, 1024);
                if (ret == -1)
                    ERR_EXIT("read");
                if (ret == 0)
                {
                    std::cout<<"client close"<<std::endl;
                    close(connfd);
                    event = events[i];
                    epoll_ctl(epollfd, EPOLL_CTL_DEL, connfd, &event);
                    clients.erase(std::remove(clients.begin(), clients.end(), connfd), clients.end());
                    continue;
                }
 
                std::cout<<buf;
                write(connfd, buf, strlen(buf));
            }
 
 
 
        }
 
    }
 
 
}
 
/*
正确的读法
n = 0;
while ((nread = read(fd, buf + n, BUFSIZ-1)) > 0) {
    n += nread;
}
if (nread == -1 && errno != EAGAIN) {
    perror("read error");
}
*/
 
 
/*
正确的写法
int nwrite, data_size = strlen(buf);
n = data_size;
while (n > 0) {
    nwrite = write(fd, buf + data_size - n, n);
    if (nwrite < n) {
        if (nwrite == -1 && errno != EAGAIN) {
            perror("write error");
        }
        break;
    }
    n -= nwrite;
}
*/
```

### epoll summary

下面的总结参照于c++教育网


- EPOLLIN事件
内核中的`socket`接收缓冲区为空-低电平

内核中的`socket`接收缓冲区不为空-高电平


- `EPOLLOUT`事件
内核中的`socket`发送缓冲区不满-高电平

内核中的`socket`发送缓冲区满-低电平


- LT电平触发
高电平触发
- ET 边沿触发
低电平--》高电平 触发

高电平--》低电平 触发  （注，本人不赞同这种观点）

 
- EPOLL 在有些情况下是不建议使用的

如：已连接套接字不多，并且这些套接字非常活跃

因为`epoll`内部的实现比较复杂（使用`callback`原理），编写时也需更加复杂的代码逻辑


### Epoll在LT和ET模式下的读写方式

在一个非阻塞的`socket`上调用`read/write`函数, 返回`EAGAIN`或者`EWOULDBLOCK`(注: `EAGAIN`就是E`WOULDBLOCK`)

从字面上看, 意思是:
- EAGAIN: 再试一次
- EWOULDBLOCK: 如果这是一个阻塞socket, 操作将被block
- perror输出:  Resource temporarily unavailable


总结:

这个错误表示资源暂时不够, 可能`read`时, 读缓冲区没有数据, 或者, `write`时,写缓冲区满了.  遇到这种情况, 如果是阻塞`socket`, `read/write`就要阻塞掉.而如果是非阻塞`socket`, `read/write`立即返回`-1`, 同时`errno`设置为`EAGAIN`.所以, 对于阻塞socket, read/write返回-1代表网络出错了.

但对于非阻塞`socket`,`read/write`返回`-1`不一定网络真的出错了.可能是`Resource temporarily unavailable`. 这时你应该再试, 直到`Resource available`.

综上, 对于`non-blocking`的`socket`,  正确的读写操作为:
- 读: 忽略掉`errno = EAGAIN`的错误, 下次继续读　
- 写: 忽略掉`errno = EAGAIN`的错误, 下次继续写　

对于`select`和`epoll`的`LT`模式, 这种读写方式是没有问题的. 但对于`epoll`的`ET`模式, 这种方式还有漏洞.

`epoll`的两种模式`LT`和`ET`

二者的差异在于`level-trigger`模式下只要某个`socket`处于`readable/writable`状态，无论什么时候进行`epoll_wait`都会返回该`socket`；而`edge-trigger`模式下只有某个`socket`从`unreadable`变为`readable`或从`unwritable`变为`writable`时`epoll_wait`才会返回该`socket`。如下两个示意图:

从`socket`读数据:

<center>![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/muduo_base_library_2.png)</center>

往`socket`写数据

<center>![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/muduo_base_library_3.png)</center>

所以, 在`epoll`的`ET`模式下, 正确的读写方式为:
- 读: 只要可读, 就一直读, 直到返回`0`, 或者`errno = EAGAIN`
- 写: 只要可写, 就一直写, 直到数据发送完, 或者`errno = EAGAIN`

正确的读:

``` c++

n = 0;  
while ((nread = read(fd, buf + n, BUFSIZ-1)) > 0) {  
    n += nread;  
}  
if (nread == -1 && errno != EAGAIN) {  
    perror("read error");  
}  

```

正确的写:

``` c++

int nwrite, data_size = strlen(buf);  
n = data_size;  
while (n > 0) {  
    nwrite = write(fd, buf + data_size - n, n);  
    if (nwrite < n) {  
        if (nwrite == -1 && errno != EAGAIN) {  
            perror("write error");  
        }  
        break;  
    }  
    n -= nwrite;  
}  
```

正确的`accept`，`accept`要考虑`2`个问题

- 阻塞模式 accept 存在的问题
考虑这种情况：`TCP`连接被客户端夭折，即在服务器调用`accept`之前，客户端主动发送`RST`终止连接，导致刚刚建立的连接从就绪队列中移出，如果套接口被设置成阻塞模式，服务器就会一直阻塞,在`accept` 调用上，直到其他某个客户建立一个新的连接为止。但是在此期间，服务器单纯地阻塞在`accept` 调用上，就绪队列中的其他描述符都得不到处理.

解决办法是把监听套接口设置为非阻塞，当客户在服务器调用`accept`之前中止某个连接时,`accept`调用可以立即返回`-1`，这时源自`Berkeley`的实现会在内核中处理该事件，并不会将该事件通知给`epool`，而其他实现把`errno`设置为`ECONNABORTED`或者`EPROTO` 错误，我们应该忽略这两个错误。

- `ET`模式下`accept`存在的问题
考虑这种情况：多个连接同时到达，服务器的`TCP`就绪队列瞬间积累多个就绪连接，由于是边缘触发模式，`epoll`只会通知一次，`accept`只处理一个连接，导致`TCP`就绪队列中剩下的连接都得不到处理。

解决办法是用`while`循环抱住`accept`调用，处理完`TCP`就绪队列中的所有连接后再退出循环。如何知道是否处理完就绪队列中的所有连接呢？`accept`  返回`-1`并且`errno`设置为`EAGAIN`就表示所有连接都处理完。

综合以上两种情况，服务器应该使用非阻塞地`accept`，`accept`在`ET`模式下 的正确使用方式为：

``` c++

while ((conn_sock = accept(listenfd,(struct sockaddr *) &remote,   
                (size_t *)&addrlen)) > 0) {  
    handle_client(conn_sock);  
}  
if (conn_sock == -1) {  
    if (errno != EAGAIN && errno != ECONNABORTED   
            && errno != EPROTO && errno != EINTR)   
        perror("accept");  
}  
 
```

一道腾讯后台开发的面试题

使用`Linux epoll`模型，水平触发模式；当`socket`可写时，会不停的触发`socket`可写的事件，如何处理？

第一种最普遍的方式：

需要向`socket`写数据的时候才把`socket`加入`epoll`，等待可写事件。接受到可写事件后，调用`write`或者`send`发送数据。当所有数据都写完后，把`socket`移出`epoll`。
这种方式的缺点是，即使发送很少的数据，也要把`socket`加入`epoll`,写完后在移出`epoll`，有一定操作代价。

一种改进的方式：
开始不把`socket`加入`epoll`，需要向`socket`写数据的时候，直接调用`write`或者`send`发送数据。
如果返回`EAGAIN`，把`socket`加入`epoll`，在`epoll`的驱动下写数据，全部数据发送完毕后，再移出`epoll`。

这种方式的优点是：数据不多的时候可以避免`epoll`的事件处理，提高效率。
最后贴一个使用`epoll`, `ET`模式的简单`HTTP`服务器代码:


``` c++
#include <sys/socket.h>
#include <sys/wait.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <sys/epoll.h>
#include <sys/sendfile.h>
#include <sys/stat.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <strings.h>
#include <fcntl.h>
#include <errno.h> 
 
#define MAX_EVENTS 10
#define PORT 8080
 
//设置socket连接为非阻塞模式
void setnonblocking(int sockfd) {
    int opts;
 
    opts = fcntl(sockfd, F_GETFL);
    if(opts < 0) {
        perror("fcntl(F_GETFL)\n");
        exit(1);
    }
    opts = (opts | O_NONBLOCK);
    if(fcntl(sockfd, F_SETFL, opts) < 0) {
        perror("fcntl(F_SETFL)\n");
        exit(1);
    }
}
 
int main(){
    struct epoll_event ev, events[MAX_EVENTS];
    int addrlen, listenfd, conn_sock, nfds, epfd, fd, i, nread, n;
    struct sockaddr_in local, remote;
    char buf[BUFSIZ];
 
    //创建listen socket
    if( (listenfd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        perror("sockfd\n");
        exit(1);
    }
    setnonblocking(listenfd);
    bzero(&local, sizeof(local));
    local.sin_family = AF_INET;
    local.sin_addr.s_addr = htonl(INADDR_ANY);;
    local.sin_port = htons(PORT);
    if( bind(listenfd, (struct sockaddr *) &local, sizeof(local)) < 0) {
        perror("bind\n");
        exit(1);
    }
    listen(listenfd, 20);
 
    epfd = epoll_create(MAX_EVENTS);
    if (epfd == -1) {
        perror("epoll_create");
        exit(EXIT_FAILURE);
    }
 
    ev.events = EPOLLIN;
    ev.data.fd = listenfd;
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, listenfd, &ev) == -1) {
        perror("epoll_ctl: listen_sock");
        exit(EXIT_FAILURE);
    }
 
    for (;;) {
        nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        if (nfds == -1) {
            perror("epoll_pwait");
            exit(EXIT_FAILURE);
        }
 
        for (i = 0; i < nfds; ++i) {
            fd = events[i].data.fd;
            if (fd == listenfd) {
                while ((conn_sock = accept(listenfd,(struct sockaddr *) &remote, 
                                (size_t *)&addrlen)) > 0) {
                    setnonblocking(conn_sock);
                    ev.events = EPOLLIN | EPOLLET;
                    ev.data.fd = conn_sock;
                    if (epoll_ctl(epfd, EPOLL_CTL_ADD, conn_sock,
                                &ev) == -1) {
                        perror("epoll_ctl: add");
                        exit(EXIT_FAILURE);
                    }
                }
                if (conn_sock == -1) {
                    if (errno != EAGAIN && errno != ECONNABORTED 
                            && errno != EPROTO && errno != EINTR) 
                        perror("accept");
                }
                continue;
            }  
            if (events[i].events & EPOLLIN) {
                n = 0;
                while ((nread = read(fd, buf + n, BUFSIZ-1)) > 0) {
                    n += nread;
                }
                if (nread == -1 && errno != EAGAIN) {
                    perror("read error");
                }
                ev.data.fd = fd;
                ev.events = events[i].events | EPOLLOUT;
                if (epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev) == -1) {
                    perror("epoll_ctl: mod");
                }
            }
            if (events[i].events & EPOLLOUT) {
                sprintf(buf, "HTTP/1.1 200 OK\r\nContent-Length: %d\r\n\r\nHello World", 11);
                int nwrite, data_size = strlen(buf);
                n = data_size;
                while (n > 0) {
                    nwrite = write(fd, buf + data_size - n, n);
                    if (nwrite < n) {
                        if (nwrite == -1 && errno != EAGAIN) {
                            perror("write error");
                        }
                        break;
                    }
                    n -= nwrite;
                }
                close(fd);
            }
        }
    }
 
    return 0;
}

```
