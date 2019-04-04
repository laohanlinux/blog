---
title: muduo_base_library_2
date: 2013-06-10 12:43:39
tags:
- muduo
- muduo base library
categories:
- muduo 
- network program
description: muduo 是一个linux 网络库，使用poll和epoll模式。 这些内容是本人在大学的时候，通过  << c++教程网 >>  所学的知识笔记. 已经过去好几年了，版本也是比较老的了，属于入门级别的教程； 如有错误，纯属正常!!!

---

## [1] Timestamp

计算机从`1970-01-01`开始时间计算 ， 但目前的经过的时间秒 假设等于`microSecondsSinceEpoch_` ， 把`microSecondsSinceEpoch_`转为`time_t`（大整数），在有`time_t`

转为`tm`就可得到准确的时间格式

``` c++
struct tm {
	int tm_sec; /* 秒–取值区间为[0,59] */

	int tm_min; /* 分 - 取值区间为[0,59] */

	int tm_hour; /* 时 - 取值区间为[0,23] */

	int tm_mday; /* 一个月中的日期 - 取值区间为[1,31] */

	int tm_mon; /* 月份（从一月开始，0代表一月） - 取值区间为[0,11] */

	int tm_year; /* 年份，其值从1900开始 */

	int tm_wday; /* 星期–取值区间为[0,6]，其中0代表星期天，1代表星期一，以此类推 */

	int tm_yday; /* 从每年的1月1日开始的天数–取值区间为[0,365]，其中0代表1月1日，1代表1月2日，以此类推 */

	int tm_isdst; /* 夏令时标识符，实行夏令时的时候，tm_isdst为正。不实行夏令时的进候，tm_isdst为0；不了解情况时，tm_isdst()为负。*/

	long int tm_gmtoff; /*指定了日期变更线东面时区中UTC东部时区正秒数或UTC西部时区的负秒数*/

	const char *tm_zone; /*当前时区的名字(与环境变量TZ有关)*/
};

```

跨平台`PRId64`的使用: `int64_t`用来表示`64`位整数，在`32`位系统中是`long long int`，在`64`位系统中是`long int`,所以打印`int64_t`的格式化方法是：

``` c++

printf(“%ld”, value);  // 64bit OS
printf("%lld", value); // 32bit OS
```

跨平台的做法：

`#define __STDC_FORMAT_MACROS`
`#include <inttypes.h>`
`#undef __STDC_FORMAT_MACROS` 
`printf("%" PRId64 "\n", value)`    
 

## [11] gcc 原子操作

// 原子自增操作
`type __sync_fetch_and_add(type *ptr, type value)`

// 原子比较和交换（设置）操作
`type __sync_val_compare_and_swap(type *ptr, type oldval type newval) `
`bool __sync_bool_compare_and_swap(type *ptr, type oldval type newval) `

// 原子赋值操作
`type __sync_lock_test_and_set (type, type value)`

无锁队列的实现： http://coolshell.cn/articles/8239.html

## [12] Exception类实现

Exception 类图

``` c++
-message_: string
 
-stack_:string
 
<<create>>-Exception(what:char)
 
<<create>>-Exception(what:string)
 
<<destroy>>-Exception()
 
+what():const char *
 
+stackTrace():const char*
 
-fillStackTrace ():void

```

下面是源代码：

头文件：

``` c++
#ifndef MUDUO_BASE_EXCEPTION_H
#define MUDUO_BASE_EXCEPTION_H
 
#include <muduo/base/Types.h>
#include <exception>
 
namespace muduo
{
 
class Exception : public std::exception
{
 public:
  explicit Exception(const char* what);
  explicit Exception(const string& what);
  virtual ~Exception() throw();
  virtual const char* what() const throw();
  const char* stackTrace() const throw();
 
 private:
  void fillStackTrace();
 
  string message_;
  string stack_;
};
 
}
 
#endif  // MUDUO_BASE_EXCEPTION_
```
源文件:

``` c++
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
//
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/base/Exception.h>
 
//#include <cxxabi.h>
#include <execinfo.h>
#include <stdlib.h>
 
using namespace muduo;
 
Exception::Exception(const char* msg)
  : message_(msg)
{
  fillStackTrace();
}
 
Exception::Exception(const string& msg)
  : message_(msg)
{
  fillStackTrace();
}
 
Exception::~Exception() throw ()
{
}
 
const char* Exception::what() const throw()
{
  return message_.c_str();
}
 
const char* Exception::stackTrace() const throw()
{
  return stack_.c_str();
}
 
//填充错误堆栈
void Exception::fillStackTrace()
{
  const int len = 200;
  void* buffer[len];
  /*
   backtrace()  returns  a  backtrace  for the calling program, in the array pointed to by buffer.  A backtrace is the series of currently
   active function calls for the program.  Each item in the array pointed to by buffer is of type void *, and is the return  address  from
   the corresponding stack frame.  The size argument specifies the maximum number of addresses that can be stored in buffer.
  */
  //buffer 存地就是函数的地址
  int nptrs = ::backtrace(buffer, len);
  /*
            Given  the  set  of  addresses returned by backtrace() in buffer, backtrace_symbols() translates the addresses into an array of strings
        that describe the addresses symbolically.  The size argument specifies the number of addresses in buffer.  The symbolic  representation
       of  each  address  consists  of  the  function name (if this can be determined), a hexadecimal offset into the function, and the actual
       return address (in hexadecimal).  The address of the array of string pointers is returned as  the  function  result  of  backtrace_sym-
       bols().   This  array  is malloc(3)ed by backtrace_symbols(), and must be freed by the caller.  (The strings pointed to by the array of
       pointers need not and should not be freed.)
 
Pointer--------pointer1 ------string
--------------pointer2 ------string
--------------pointer3 ------string
--------------pointer4 ------string
--------------pointer5 ------string
--------------pointer6 ------string
 
我们要释放pointer占用的内存，但是pointer所指向的内存不需要我们释放
free (Pointer) 就OK了
  */
  //将buffer里面的地址转换为函数符号，也可以所说函数的名字
  char** strings = ::backtrace_symbols(buffer, nptrs);
  if (strings)
  {
    for (int i = 0; i < nptrs; ++i)
    {
      // TODO demangle funcion name with abi::__cxa_demangle
      stack_.append(strings[i]);
      stack_.push_back('\n');
    }
    //这个要自己释放
    free(strings);
  }
}

```

测试程序:
 
``` c++
#include <muduo/base/Exception.h>
#include <stdio.h>
 
class Bar
{
 public:
  void test()
  {
    throw muduo::Exception("oops");
  }
};
 
void foo()
{
  Bar b;
  b.test();
}
 
int main()
{
  try
  {
    foo();
  }
  catch (const muduo::Exception& ex)
  {
    printf("reason: %s\n", ex.what());
    printf("stack trace: %s\n", ex.stackTrace());
  }
}
```

程序输出：

``` sh
reason: oops
stack trace: muduo::Exception::fillStackTrace()
muduo::Exception::Exception(char const*)
Bar::test()
foo()
./exception_test(main+0x10)
/lib/libc.so.6(__libc_start_main+0xe6)
./exception_test()
```

## [13] muduo 之线程

### 线程标识符

``` c++
    pthread_self gettid
    __thread，gcc内置的线程局部存储设施,每个线程都有一份
    __thread只能修饰POD类型
POD类型（plain old data），与C兼容的原始数据，例如，结构和整型等C语言中的类型是 POD 类型，但带有用户定义的构造函数或虚函数的类则不是
    __thread string t_obj1(“cppcourse”);    // 错误，不能调用对象的构造函数
    __thread string* t_obj2 = new string;    // 错误，初始化只能是编译期常量
    __thread string* t_obj3 = NULL;    // 正确
    boost::is_same
    pthread_atfork
```

- Linux中，每个进程有一个`pid`，类型`pid_t`，由`getpid()`取得。Linux下的`POSIX`线程也有一个`id`，类型 `pthread_t`，由`pthread_self()`取得，该id由线程库维护，其`id`空间是各个进程独立的（即不同进程中的线程可能有相同的`id`）。`Linux`中的`POSIX`线程库实现的线程其实也是一个进程`（LWP）`，只是该进程与主进程（启动线程的进程）共享一些资源而已，比如代码段，数据段等。
- 有时候我们可能需要知道线程的真实`pid`。比如进程`P1`要向另外一个进程`P2`中的某个线程发送信号时，既不能使用P2的`pid`，更不能使用线程的`pthread id`，而只能使用该线程的真实`pid`，称为`tid`。
- 有一个函数`gettid()`可以得到`tid`，但`glibc`并没有实现该函数，只能通过`Linux`的系统调用`syscall`来获取。`return syscall(SYS_gettid)`

### muduo库的Thread类图

`typedef boost::function<void ()> ThreadFunc;`

``` c++
-started_:bool
 
-pthreadId : pthread_t
 
-tid_:pid_t//
 
-func_:ThreadFunc
 
-name_:string //线程的名称
 
-numCreated_:AtomicInt32 //已经创建线程的个数
 
-------------------------------
 
<<create>>+Thread(func:const ThreadFunc&,name:string)
 
<<destroy>>+Thread()
 
+start():void
 
+join():int
 
+started():bool //线程是否已经启动了
 
+tid():pid_t
 
+name():const string &
 
+numCreated():int //已近启动线程的个数
 
-startThread(thread:void):void *
 
-runInThread():void
 
-----------------------------------------------------------------------
 
函数调用关系
 
startThread()--->runInthread--->func
```

*源文件*:

头文件：Thread.h

``` go
namespace muduo
{
 
class Thread : boost::noncopyable
{
 public:
  typedef boost::function<void ()> ThreadFunc;
 
  explicit Thread(const ThreadFunc&, const string& name = string());
  ~Thread();
 
  void start();
  int join(); // return pthread_join()
 
  bool started() const { return started_; }
  // pthread_t pthreadId() const { return pthreadId_; }
  pid_t tid() const { return tid_; }
  const string& name() const { return name_; }
 
  static int numCreated() { return numCreated_.get(); }
 
 private:
  static void* startThread(void* thread);
  void runInThread();
 
  bool       started_;
  pthread_t  pthreadId_;
  pid_t      tid_;
  ThreadFunc func_;
  string     name_;
 
  static AtomicInt32 numCreated_;
};
 
}
#endif
```

源文件：Thread.cpp

``` c++
namespace muduo
{
namespace CurrentThread
{ // __thread修饰的变量时线程局部存储的。 也就是所个线程都一份
  __thread int t_cachedTid = 0; //线程真实pid(tid)的缓存，如果每次都使用syscall(SYS_gettid)来获取tid，那么效率会很低
  __thread char t_tidString[32];//这是tid的字符串表示形式
  __thread const char* t_threadName = "unknown";
  const bool sameType = boost::is_same<int, pid_t>::value;
  BOOST_STATIC_ASSERT(sameType);
}
 
namespace detail
{
 
pid_t gettid()
{
  return static_cast<pid_t>(::syscall(SYS_gettid));
}
 
void afterFork()
{
  muduo::CurrentThread::t_cachedTid = 0;
  muduo::CurrentThread::t_threadName = "main";
  CurrentThread::tid();
  // no need to call pthread_atfork(NULL, NULL, &afterFork);
}
 
class ThreadNameInitializer
{
 public:
  ThreadNameInitializer()
  {
    muduo::CurrentThread::t_threadName = "main";
    CurrentThread::tid();
    pthread_atfork(NULL, NULL, &afterFork);
  }
};
 
ThreadNameInitializer init;
}
}
 
using namespace muduo;
 
void CurrentThread::cacheTid()
{
  if (t_cachedTid == 0)
  {
    t_cachedTid = detail::gettid();
    int n = snprintf(t_tidString, sizeof t_tidString, "%5d ", t_cachedTid);
    assert(n == 6); (void) n;
  }
}
 
bool CurrentThread::isMainThread()
{
  return tid() == ::getpid();
}
 
AtomicInt32 Thread::numCreated_;
 
Thread::Thread(const ThreadFunc& func, const string& n)
  : started_(false),
    pthreadId_(0),
    tid_(0),
    func_(func),
    name_(n)
{
  numCreated_.increment();
}
 
Thread::~Thread()
{
  // no join
}
 
void Thread::start()
{
  assert(!started_);
  started_ = true;
  errno = pthread_create(&pthreadId_, NULL, &startThread, this);
  if (errno != 0)
  {
    LOG_SYSFATAL << "Failed in pthread_create";
  }
}
 
int Thread::join()
{
  assert(started_);
  return pthread_join(pthreadId_, NULL);
}
 
void* Thread::startThread(void* obj)
{
  Thread* thread = static_cast<Thread*>(obj);
  thread->runInThread();
  return NULL;
}
 
void Thread::runInThread()
{
  tid_ = CurrentThread::tid();
  muduo::CurrentThread::t_threadName = name_.c_str();
  try
  {
    func_();
    muduo::CurrentThread::t_threadName = "finished";
  }
  catch (const Exception& ex)
  {
    muduo::CurrentThread::t_threadName = "crashed";
    fprintf(stderr, "exception caught in Thread %s\n", name_.c_str());
    fprintf(stderr, "reason: %s\n", ex.what());
    fprintf(stderr, "stack trace: %s\n", ex.stackTrace());
    abort();
  }
  catch (const std::exception& ex)
  {
    muduo::CurrentThread::t_threadName = "crashed";
    fprintf(stderr, "exception caught in Thread %s\n", name_.c_str());
    fprintf(stderr, "reason: %s\n", ex.what());
    abort();
  }
  catch (...)
  {
    muduo::CurrentThread::t_threadName = "crashed";
    fprintf(stderr, "unknown exception caught in Thread %s\n", name_.c_str());
    throw; // rethrow
  }
}

```

测试程序1 

``` php
#include <muduo/base/Thread.h>
#include <muduo/base/CurrentThread.h>
 
#include <string>
#include <boost/bind.hpp>
#include <stdio.h>
 
void threadFunc()
{
  printf("tid=%d\n", muduo::CurrentThread::tid());
}
 
void threadFunc2(int x)
{
  printf("tid=%d, x=%d\n", muduo::CurrentThread::tid(), x);
}
 
class Foo
{
 public:
  explicit Foo(double x)
    : x_(x)
  {
  }
 
  void memberFunc()
  {
    printf("tid=%d, Foo::x_=%f\n", muduo::CurrentThread::tid(), x_);
  }
 
  void memberFunc2(const std::string& text)
  {
    printf("tid=%d, Foo::x_=%f, text=%s\n", muduo::CurrentThread::tid(), x_, text.c_str());
  }
 
 private:
  double x_;
};
 
int main()
{
  printf("pid=%d, tid=%d\n", ::getpid(), muduo::CurrentThread::tid());
 
  muduo::Thread t1(threadFunc);
  t1.start();
  t1.join();
 
  muduo::Thread t2(boost::bind(threadFunc2, 42),
                   "thread for free function with argument");
  t2.start();
  t2.join();
 
  Foo foo(87.53);
  muduo::Thread t3(boost::bind(&Foo::memberFunc, &foo),
                   "thread for member function without argument");
  t3.start();
  t3.join();
 
  muduo::Thread t4(boost::bind(&Foo::memberFunc2, boost::ref(foo), std::string("Shuo Chen")));
  t4.start();
  t4.join();
 
  printf("number of created threads %d\n", muduo::Thread::numCreated());
```

程序输出：

``` sh
[root@localhost bin]# ./thread_test
pid=3133, tid=3133
tid=3134
tid=3135, x=42
tid=3136, Foo::x_=87.530000
tid=3137, Foo::x_=87.530000, text=Shuo Chen
number of created threads 4
[root@localhost bin]# ^C
```

### pthread_atfork

``` c++
#include <pthread.h>
int pthread_atfork(void (*prepare)(void), void (*parent)(void), void (*child)(void));
```
调用fork时，内部创建子进程前在父进程中会调用prepare，内部创建子进程成功后，父进程会调用parent ，子进程会调用child.

测试源码：

``` c++
#include <stdio.h>
#include <time.h>
#include <pthread.h>
#include <unistd.h>
 
void prepare(void)
{
    printf("pid = %d prepare ...\n", static_cast<int>(getpid()));
}
 
void parent(void)
{
    printf("pid = %d parent ...\n", static_cast<int>(getpid()));
}
 
void child(void)
{
    printf("pid = %d child ...\n", static_cast<int>(getpid()));
}
 
 
int main(void)
{
    printf("pid = %d Entering main ...\n", static_cast<int>(getpid()));
 
    pthread_atfork(prepare, parent, child);
 
    fork();
 
    printf("pid = %d Exiting main ...\n",static_cast<int>(getpid()));
 
    return 0;
}
```

程序输出：

``` sh
[root@localhost bin]# ./pthread_atfork_test
pid = 3358 Entering main ...
pid = 3358 prepare ...
pid = 3358 parent ...
pid = 3358 Exiting main ...
[root@localhost bin]# pid = 3359 child ...
pid = 3359 Exiting main ...
 
[root@localhost bin]# ^C
```

注意：

在编程时要特别注意，不要使用多线程和多进程进行交叉使用，很有可能出现死锁的情况！！！

## [13-1] module之线程

``` c++
#include<iostream>
#include<pthread.h>
#include<unistd.h>
 
using namespace std;
const int i = 5 ;
 
__thread int var = i ;
 
void * worker1(void * args);
void * worker2(void * args);
 
 
int main(void){
    pthread_t pid1, pid2;
    static __thread int temp = 10 ;
 
    pthread_create(&pid1, NULL, worker1, NULL);
    pthread_create(&pid2, NULL, worker2, NULL);
 
    pthread_join(pid1, NULL);
    pthread_join(pid2, NULL);
    cout<<temp<<endl;
    return 0;
}
 
 
void * worker1(void *args){
    cout<<++var<<endl;
    cout<<++var<<endl;
 
}
void * worker2(void *args){
    sleep(3);
    cout<<++var<<endl;
}
```

程序输出：

``` sh 
oem@oem-VirtualBox ~/workspace/c++ $ ./a.out
6
7
6
10
oem@oem-VirtualBox ~/workspace/c++ $
```

`_thread`是`GCC`内置的线程局部存储设施，存取效率可以和全局变量相比。`__thread`变量每一个线程有一份独立实体，各个线程的值互不干扰。可以用来修饰那些带有全局性且值可能变，但是又不值得用全局变量保护的变量。

`__thread`使用规则：只能修饰`POD`类型(类似整型指针的标量，不带自定义的构造、拷贝、赋值、析构的类型，二进制内容可以任意复制`memset`,`memcpy`,且内容可以复原)，不能修饰`class`类型，因为无法自动调用构造函数和析构函数，可以用于修饰全局变量，函数内的静态变量，不能修饰函数的局部变量或者`class`的普通成员变量，且`__thread`变量值只能初始化为编译器常量(值在编译器就可以确定`const int i=5`,运行期常量是运行初始化后不再改变`const int i=rand())`.


## [14] IPC组件的分装之互斥锁

### MutexLock 的封装

- MutexLock类图

``` c++
-mutex_:pthread_mutex_t 

-holder_:pid_t //线程的真实ID,用来表示拥有锁的线程

-----------------------------

<<create>>-MutexLock()

<<destroy>>-MutexLock

+isLockedByThisThread:bool // 当前线程是否拥有该锁

+assertLocked():void //断言当前线程是否拥有该锁

+lock():void

+unlock():void

+getPthreadMutex():pthread_mutex_t *
```

- MutexLockGuard类图封装:

``` c++
-mutex_:MutexLock &

-----------

<<create>>-MutexLockGurad(mutex:MutexLock&)

<<destroy>>-MutexLockGurad()

```

### MutexLockGuard

- MutexLock类图

``` c++
-mutex_:pthread_mutex_t 

-holder_:pid_t //线程的真实ID,用来表示拥有锁的线程

-----------------------------

<<create>>-MutexLock()

<<destroy>>-MutexLock

+isLockedByThisThread:bool // 当前线程是否拥有该锁

+assertLocked():void //断言当前线程是否拥有该锁

+lock():void

+unlock():void

+getPthreadMutex():pthread_mutex_t *

```

- MutexLockGuard类图封装

``` c++
-mutex_:MutexLock &

-----------

<<create>>-MutexLockGurad(mutex:MutexLock&)

<<destroy>>-MutexLockGurad()
```
### MutexLockGuard

- MutexLockGuard 类图

``` c++
-mutex_:MutexLock&

<<create>>-MutexLockGuard(mutex:MutexLock&)

<<destroy>>-MutexLockGuard()
```

### MutexLock 和 MutexLockGuard的源代码

``` go
/ Use of this source code is governed by a BSD-style license
// that can be found in the License file.
//
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#ifndef MUDUO_BASE_MUTEX_H
#define MUDUO_BASE_MUTEX_H
 
#include <muduo/base/CurrentThread.h>
#include <boost/noncopyable.hpp>
#include <assert.h>
#include <pthread.h>
 
namespace muduo
{
 
class MutexLock : boost::noncopyable //表示该类不能复制
{
 public:
  MutexLock()
    : holder_(0)
  {
     //初始化锁
    int ret = pthread_mutex_init(&mutex_, NULL);
    assert(ret == 0); (void) ret;
  }
 
  ~MutexLock()
  {
      //断言该锁是否还在使用
    assert(holder_ == 0);
    //没用呗使用则销毁锁资源
    int ret = pthread_mutex_destroy(&mutex_);
    assert(ret == 0); (void) ret;
  }
 //测试当前线程是否拥有锁
  bool isLockedByThisThread()
  {
    return holder_ == CurrentThread::tid();
  }
//断言锁
  void assertLocked()
  {
    assert(isLockedByThisThread());
  }
 
  // internal usage
//当前线程枷锁
  void lock()
  {
    pthread_mutex_lock(&mutex_);
    holder_ = CurrentThread::tid();
  }
//当前线程解锁
  void unlock()
  {
    holder_ = 0;
    pthread_mutex_unlock(&mutex_);
  }
 
  pthread_mutex_t* getPthreadMutex() /* non-const */
  {
    return &mutex_;
  }
 
 private:
 
  pthread_mutex_t mutex_;
  pid_t holder_;
};
 
 
class MutexLockGuard : boost::noncopyable //不可复制
{
 public:
//构造时负责枷锁
  explicit MutexLockGuard(MutexLock& mutex)
    : mutex_(mutex)
  {
    mutex_.lock();
  }
//虚构时负责解锁，但是并没有销毁mutex_
  ~MutexLockGuard()
  {
    mutex_.unlock();
  }
 
 private:
// MutexLock &mutex 的生存期并不会被MutexLockGuard管理
 //他们的关系是关联的。
 // 聚合关系：表示A包含B
 // 组合关系：表示A包含B，并且负责B的生存期
  MutexLock& mutex_;
};
 
}
 
// Prevent misuse like:
// MutexLockGuard(mutex_); 防止匿名对象
// A tempory object doesn't hold the lock for long!
#define MutexLockGuard(x) error "Missing guard object name"
 
#endif  // MUDUO_BA
```

测试代码:

``` go
#include <muduo/base/CountDownLatch.h>
#include <muduo/base/Mutex.h>
#include <muduo/base/Thread.h>
#include <muduo/base/Timestamp.h>
#include <boost/bind.hpp>
#include <boost/ptr_container/ptr_vector.hpp>
#include <vector>
#include <stdio.h>
 
using namespace muduo;
using namespace std;
 
MutexLock g_mutex;
vector<int> g_vec;
const int kCount = 10*1000*1000;
 
void threadFunc()
{
    //插入kCount个整数
  for (int i = 0; i < kCount; ++i)
  {
      //没循环一次加解锁一次，这里花费的时间会多点
    MutexLockGuard lock(g_mutex);
    g_vec.push_back(i);
  }
}
 
int main()
{
    //最多8 个线程
  const int kMaxThreads = 8;、
    //预留8千多万个整数 ， 300多m
  g_vec.reserve(kMaxThreads * kCount);
    //登记一个时间戳
  Timestamp start(Timestamp::now());
    //插入kCount多个整数
  for (int i = 0; i < kCount; ++i)
  {
    g_vec.push_back(i);
  }
    //单线程 统计插入kCount个整数所用时间
  printf("single thread without lock %f\n", timeDifference(Timestamp::now(), start));
 
  start = Timestamp::now();
  threadFunc();
  printf("single thread with lock %f\n", timeDifference(Timestamp::now(), start));
 
  for (int nthreads = 1; nthreads < kMaxThreads; ++nthreads)
  {
    //这是一个指针vector，存放的是Thread乐行的指针
    //智能指针，释放threads时，它里面的对象也会被销毁
    boost::ptr_vector<Thread> threads;
    g_vec.clear();
    start = Timestamp::now();
    /*
    第一次 1 个线程， 第二次2个线程.....，不同线程个数时所花费的时间
    **/
    for (int i = 0; i < nthreads; ++i)
    {
      threads.push_back(new Thread(&threadFunc));
      //启动最后一个对象的线程，其实就是每增加一个，然后就执行它
      threads.back().start();
    }
    //等待线程的结束
    for (int i = 0; i < nthreads; ++i)
    {
      threads[i].join();
    }
    //计算时间差
    printf("%d thread(s) with lock %f\n", nthreads, timeDifference(Timestamp::now(), start));
  }
}

```

程序输出:

`single thread without lock 1.512640`
`single thread with lock 1.305807`
`1 thread(s) with lock 0.685571`
`2 thread(s) with lock 3.381297`
`3 thread(s) with lock 4.640776`
`4 thread(s) with lock 5.383397`
`5 thread(s) with lock 7.897223`
`6 thread(s) with lock 21.798079`
`7 thread(s) with lock 23.351875`

## [15] BlockingQueue


### BlockingQueue类图


``` c++
-mutex_ : MutexLock //互斥锁    

-notEmpty : Condition //            

-queue_: std::deque<T> //堆栈  

---------------------------            

<<create>>-BlockingQueue()    

+put(X:const T&):void                

+take():T                                    

+size():size_t                                

```

### BlockingQueue的源文件:

``` go
// that can be found in the License file.
//
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#ifndef MUDUO_BASE_BLOCKINGQUEUE_H
#define MUDUO_BASE_BLOCKINGQUEUE_H
 
#include <muduo/base/Condition.h>
#include <muduo/base/Mutex.h>
 
#include <boost/noncopyable.hpp>
#include <deque>
#include <assert.h>
 
/**
阻塞队列，无界限队列
 
**/
namespace muduo
{
 
template<typename T>  //模板
class BlockingQueue : boost::noncopyable //不可复制的
{
 public:
  BlockingQueue() //构造函数
    : mutex_(), //互斥量
      notEmpty_(mutex_), // 测试队列是否为空，在测试直线先加锁
      queue_()  // 队列
  {
  }
 
    //人队列
  void put(const T& x)
  {
      //先加锁
    MutexLockGuard lock(mutex_);
    //入栈
    queue_.push_back(x);
    //发送信号
    notEmpty_.notify(); // TODO: move outside of lock
  }
    //出队列
  T take()
  {
      //出栈前先加锁，防止出错
    MutexLockGuard lock(mutex_);
    // always use a while-loop, due to spurious wakeup
    //如果队列为空，一直等待。。。。
    while (queue_.empty())
    {
      notEmpty_.wait();
    }
    //断言队列是否为空
    assert(!queue_.empty());
    //第一个元素出队列
    T front(queue_.front());
    queue_.pop_front();
    return front;
  }
    //队列的大小
  size_t size() const
  {
    MutexLockGuard lock(mutex_);
    return queue_.size();
  }
 
 private:
     //互斥量，主要是赋给 MutexLock--->MutexLockGuard --->&MutexLock
  mutable MutexLock mutex_;
  Condition         notEmpty_;
  std::deque<T>     queue_;
};
 
}
 
#endif  // MUDUO_BASE_BLOCKINGQUEUE_H
```

### BlockingQueue的测试程序1:

``` c++
#include <muduo/base/BlockingQueue.h>
#include <muduo/base/CountDownLatch.h>
#include <muduo/base/Thread.h>
#include <muduo/base/Timestamp.h>
 
#include <boost/bind.hpp>
#include <boost/ptr_container/ptr_vector.hpp>
#include <map>
#include <string>
#include <stdio.h>
/*
 BlockingQueue 的测试程序
**/
class Bench
{
 public:
  Bench(int numThreads)
    : latch_(numThreads), //计算器为numThreads ，就是线程的个数
      threads_(numThreads) //线程容器的大小numThreads
  {
     //加入numThreads个线程
    for (int i = 0; i < numThreads; ++i)
    {
      char name[32];
      snprintf(name, sizeof name, "work thread %d", i);
      threads_.push_back(new muduo::Thread(
            boost::bind(&Bench::threadFunc, this), muduo::string(name)));
    }
    //启动线程
    for_each(threads_.begin(), threads_.end(), boost::bind(&muduo::Thread::start, _1));
  }
/*主线程的Run函数*/
  void run(int times)
  {
    printf("waiting for count down latch\n");
    //等待计算器为0 ，就是所其他线程都准备好好，主线程才运行 ！
    latch_.wait();
    printf("all threads started\n");
    //想队列里面加入times个时间戳
    for (int i = 0; i < times; ++i)
    {
      muduo::Timestamp now(muduo::Timestamp::now());
      queue_.put(now);
      usleep(1000);
    }
  }
 
  void joinAll()
  {
    for (size_t i = 0; i < threads_.size(); ++i)
    {
      queue_.put(muduo::Timestamp::invalid());
    }
 
    for_each(threads_.begin(), threads_.end(), boost::bind(&muduo::Thread::join, _1));
  }
 
 private:
//线程的回调函数
  void threadFunc()
  {
    printf("tid=%d, %s started\n",
           muduo::CurrentThread::tid(),
           muduo::CurrentThread::name());
 
    std::map<int, int> delays;
    //协程的计算器-1
    latch_.countDown();
    //如果时间无效，则false
    bool running = true;
    while (running)
    {
        //时间戳 t ， now
      muduo::Timestamp t(queue_.take());
      muduo::Timestamp now(muduo::Timestamp::now());
      if (t.valid())
      {
        int delay = static_cast<int>(timeDifference(now, t) * 1000000);
        // printf("tid=%d, latency = %d us\n",
        //        muduo::CurrentThread::tid(), delay);
        ++delays[delay];
      }
      running = t.valid();
    }
 
    printf("tid=%d, %s stopped\n",
           muduo::CurrentThread::tid(),
           muduo::CurrentThread::name());
    for (std::map<int, int>::iterator it = delays.begin();
        it != delays.end(); ++it)
    {
      printf("tid = %d, delay = %d, count = %d\n",
             muduo::CurrentThread::tid(),
             it->first, it->second);
    }
  }
 
  muduo::BlockingQueue<muduo::Timestamp> queue_; // 无界限缓冲区
  muduo::CountDownLatch latch_;   //线程同步类，协程
  boost::ptr_vector<muduo::Thread> threads_;  //线程数目
};
 
int main(int argc, char* argv[])
{
    // 线程的个数
  int threads = argc > 1 ? atoi(argv[1]) : 1;
 
  Bench t(threads);
  t.run(10000);
  t.joinAll();
}

```

### BlockingQueue的测试程序2:

``` go
#include <muduo/base/BlockingQueue.h>
#include <muduo/base/CountDownLatch.h>
#include <muduo/base/Thread.h>
 
#include <boost/bind.hpp>
#include <boost/ptr_container/ptr_vector.hpp>
#include <string>
#include <stdio.h>
 
class Test
{
 public:
  Test(int numThreads)
    : latch_(numThreads),
      threads_(numThreads)
  {
    for (int i = 0; i < numThreads; ++i)
    {
      char name[32];
      snprintf(name, sizeof name, "work thread %d", i);
      threads_.push_back(new muduo::Thread(
            boost::bind(&Test::threadFunc, this), muduo::string(name)));
    }
    for_each(threads_.begin(), threads_.end(), boost::bind(&muduo::Thread::start, _1));
  }
 
  void run(int times)
  {
    printf("waiting for count down latch\n");
    latch_.wait();
    printf("all threads started\n");
    for (int i = 0; i < times; ++i)
    {
      char buf[32];
      snprintf(buf, sizeof buf, "hello %d", i);
      queue_.put(buf);
      printf("tid=%d, put data = %s, size = %zd\n", muduo::CurrentThread::tid(), buf, queue_.size());
    }
  }
 
  void joinAll()
  {
    for (size_t i = 0; i < threads_.size(); ++i)
    {
        //加入线程终止条件“stop”标示
      queue_.put("stop");
    }
    //开启线程
    for_each(threads_.begin(), threads_.end(), boost::bind(&muduo::Thread::join, _1));
  }
 
 private:
 
  void threadFunc()
  {
    printf("tid=%d, %s started\n",
           muduo::CurrentThread::tid(),
           muduo::CurrentThread::name());
 
    latch_.countDown();
    bool running = true;
    while (running)
    {
      std::string d(queue_.take());
      printf("tid=%d, get data = %s, size = %zd\n", muduo::CurrentThread::tid(), d.c_str(), queue_.size());
      running = (d != "stop");
    }
 
    printf("tid=%d, %s stopped\n",
           muduo::CurrentThread::tid(),
           muduo::CurrentThread::name());
  }
 
  muduo::BlockingQueue<std::string> queue_; //队列 ，条件变量的测试量
  muduo::CountDownLatch latch_; //协程
  boost::ptr_vector<muduo::Thread> threads_; //线程个数
};
 
int main()
{
  printf("pid=%d, tid=%d\n", ::getpid(), muduo::CurrentThread::tid());
  //5个子线程
  Test t(5);
  //100容量的queue
  t.run(100);
  t.joinAll();
 
  printf("number of created threads %d\n", muduo::Thread::numCreated());
}
```

## [15-1] BoundedBlockingQueue

### BoundedBlockingQueue的类图

``` go
-mutex_: MutexLock       //  互斥量     |

-notEmppty_:Condition  //消费者发送生产者的条件变量 |

-notFull_    : Condition  //生产者发送给消费者的条件变量|

-queue_:boost::circular_buffer<T> //虚唤醒队列            |

--------------------------------------

<<create>-BoudedBlockingQueue(maxSize:int) //

+put(x:T) : void         //入队列                                        |

+take():T                    //出队列

+empty() :bool         //判断队列是否为空

+full():bool              //判断队列是否满

+size() :size_t             //产品的多少

+capacity():size_t        //仓库的容量

```
### BoudedBlockingQueue的源代码


``` go
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
//
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#ifndef MUDUO_BASE_BOUNDEDBLOCKINGQUEUE_H
#define MUDUO_BASE_BOUNDEDBLOCKINGQUEUE_H
 
#include <muduo/base/Condition.h>
#include <muduo/base/Mutex.h>
 
#include <boost/circular_buffer.hpp>
#include <boost/noncopyable.hpp>
#include <assert.h>
/**
有界缓冲区 ：
Mutex + condition + circular_buffer 实现
**/
namespace muduo
{
 
template<typename T>
class BoundedBlockingQueue : boost::noncopyable
{
 public:
  explicit BoundedBlockingQueue(int maxSize)
    : mutex_(), //互斥量
      notEmpty_(mutex_), // 测试是否为空
      notFull_(mutex_),  // 测试是否已满
      queue_(maxSize)    // 返回队列的已被使用的大小
  {
  }
 
    //把数据填充到队列中去
  void put(const T& x)
  {
      //先加锁
    MutexLockGuard lock(mutex_);
      //等待队列有空闲，直到notFull_.notify()信号的唤醒
    while (queue_.full())
    {
      notFull_.wait();
    }
    //断言队列是否空（原子操作）
    assert(!queue_.full());
    //
    queue_.push_back(x);
    //通知
    notEmpty_.notify(); // TODO: move outside of lock
  }
    //把数据从队列中提取出来
  T take()
  {
      //加锁
    MutexLockGuard lock(mutex_);
    //等待队列有数据，直到notEmpty_.notify()信号的唤醒
    while (queue_.empty())
    {
      notEmpty_.wait();
    }
    //断言队列是否为空
    assert(!queue_.empty());
    //出数据
    T front(queue_.front());
    queue_.pop_front();
    //发送notFull_.notify()信号
    notFull_.notify(); // TODO: move outside of lock
    return front;
  }
 
    //判断队列是否为空
  bool empty() const
  {
      //先加锁
    MutexLockGuard lock(mutex_);
    return queue_.empty();
  }
    //判断队列是否已满
  bool full() const
  {
      //加锁
    MutexLockGuard lock(mutex_);
    return queue_.full();
  }
    //数据的大小
  size_t size() const
  {
    MutexLockGuard lock(mutex_);
    return queue_.size();
  }
    //队列的容量大小
  size_t capacity() const
  {
    MutexLockGuard lock(mutex_);
    return queue_.capacity();
  }
 
 private:
  mutable MutexLock          mutex_;
  Condition                  notEmpty_; //两个条件变量
  Condition                  notFull_;  //
  boost::circular_buffer<T>  queue_;
};
 
}
 
#endif  // MUDUO_BASE_BOUNDEDBLOCKINGQUEUE_H
```

### 测试程序

``` python

// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
//
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#ifndef MUDUO_BASE_BOUNDEDBLOCKINGQUEUE_H
#define MUDUO_BASE_BOUNDEDBLOCKINGQUEUE_H
 
#include <muduo/base/Condition.h>
#include <muduo/base/Mutex.h>
 
#include <boost/circular_buffer.hpp>
#include <boost/noncopyable.hpp>
#include <assert.h>
/**
有界缓冲区 ：
Mutex + condition + circular_buffer 实现
函数功能，类消费者和生产者
**/
namespace muduo
{
 
template<typename T>
class BoundedBlockingQueue : boost::noncopyable
{
 public:
  explicit BoundedBlockingQueue(int maxSize)
    : mutex_(), //互斥量
      notEmpty_(mutex_), // 测试是否为空
      notFull_(mutex_),  // 测试是否已满
      queue_(maxSize)    // 返回队列的已被使用的大小
  {
  }
 
    //把数据填充到队列中去
  void put(const T& x)
  {
      //先加锁
    MutexLockGuard lock(mutex_);
      //等待队列有空闲，直到notFull_.notify()信号的唤醒
    while (queue_.full())
    {
      notFull_.wait();
    }
    //断言队列是否空（原子操作）
    assert(!queue_.full());
    //
    queue_.push_back(x);
    //通知
    notEmpty_.notify(); // TODO: move outside of lock
  }
    //把数据从队列中提取出来
  T take()
  {
      //加锁
    MutexLockGuard lock(mutex_);
    //等待队列有数据，直到notEmpty_.notify()信号的唤醒
    while (queue_.empty())
    {
      notEmpty_.wait();
    }
    //断言队列是否为空
    assert(!queue_.empty());
    //出数据
    T front(queue_.front());
    queue_.pop_front();
    //发送notFull_.notify()信号
    notFull_.notify(); // TODO: move outside of lock
    return front;
  }
 
    //判断队列是否为空
  bool empty() const
  {
      //先加锁
    MutexLockGuard lock(mutex_);
    return queue_.empty();
  }
    //判断队列是否已满
  bool full() const
  {
      //加锁
    MutexLockGuard lock(mutex_);
    return queue_.full();
  }
    //数据的大小
  size_t size() const
  {
    MutexLockGuard lock(mutex_);
    return queue_.size();
  }
    //队列的容量大小
  size_t capacity() const
  {
    MutexLockGuard lock(mutex_);
    return queue_.capacity();
  }
 
 private:
  mutable MutexLock          mutex_;
  Condition                  notEmpty_; //两个条件变量
  Condition                  notFull_;  //
  boost::circular_buffer<T>  queue_;
};
 
}
 
#endif  // MUDUO_BASE_BOUNDEDBLOCKINGQUEUE_H
```

## [16] ThreadPool

<center>![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/ThreadPool.png)</center>

### threadPool类图

``` c++
-mutex_:MutexLock  //互斥量 ---> mutexlock--->mutexlockGuard
-cond_:Codition //条件变量
-name:string    //名字
-threads_:boost::ptr_vector<muduo::Thread> //线程池容量
-queue_: std::deque<Task>    //任务队列，任务我们把他作为“函数”
-running_ :bool        // 线程池是否运行
-----------------------------
<<create>>-ThreadPool(name:string)
<<destroy>>-ThreadPool()
+start(numThreads:int):void// 启动线程池
+stop():void            //关闭线程池
+run(f:Task):void //    启动
+runInThread():void //线程池中的线程的回调函数
+take():Task         // 获取任务
```

### ThreadPool头文件

``` go
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
//
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#ifndef MUDUO_BASE_THREADPOOL_H
#define MUDUO_BASE_THREADPOOL_H
 
#include <muduo/base/Condition.h>
#include <muduo/base/Mutex.h>
 
 
#include <muduo/base/Thread.h>
#include <muduo/base/Types.h>
 
#include <boost/function.hpp>
#include <boost/noncopyable.hpp>
#include <boost/ptr_container/ptr_vector.hpp>
 
#include <deque>
 
namespace muduo
{
 
class ThreadPool : boost::noncopyable //不可复制
{
 public:
  typedef boost::function<void ()> Task;
 
  explicit ThreadPool(const string& name = string());
  ~ThreadPool();
 
  void start(int numThreads);
  void stop();
 
  void run(const Task& f);
 
 private:
  void runInThread();
  Task take();
 
  MutexLock mutex_;
  Condition cond_;
  string name_;
  boost::ptr_vector<muduo::Thread> threads_;
  std::deque<Task> queue_; //线程池的任务队列
  bool running_;//线程池是否处于启动状态
};
 
}
 
#endif
```

### ThreadPool的源文件

``` c++
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
//
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#include <muduo/base/ThreadPool.h>
 
#include <muduo/base/Exception.h>
 
#include <boost/bind.hpp>
#include <assert.h>
#include <stdio.h>
 
using namespace muduo;
 
ThreadPool::ThreadPool(const string& name)
  : mutex_(), //初始化互斥量
    cond_(mutex_), //互斥量跟条件变量进行绑定
    name_(name),
    running_(false)
{
}
//线程池的虚构
ThreadPool::~ThreadPool()
{
    //如果线程池处于运行状态，关闭线程池里的线程
  if (running_)
  {
    stop();
  }
}
//1----->>启动线程池
void ThreadPool::start(int numThreads)
{
    //断言线程池是否为空
  assert(threads_.empty());
  //启动线程池
  running_ = true;
  //开辟numThreads个容量的线程池
  threads_.reserve(numThreads);
  //生产线程，这些线程可以说就是“消费者”
  for (int i = 0; i < numThreads; ++i)
  {
    char id[32];
    snprintf(id, sizeof id, "%d", i);
    threads_.push_back(new muduo::Thread(
        //线程池里的线程的回调函数  runInThread
          boost::bind(&ThreadPool::runInThread, this), name_+id));
    //启动线程
    threads_[i].start();
  }
}
 
/**
停止线程池里的线程
***/
void ThreadPool::stop()
{
  {
      //加锁
  MutexLockGuard lock(mutex_);
  running_ = false;
    //唤醒线程
  cond_.notifyAll();
  }
  //把线程加入退出队列
  for_each(threads_.begin(),
           threads_.end(),
           boost::bind(&muduo::Thread::join, _1));
}
//4---->>> 启动线程池
void ThreadPool::run(const Task& task)
{
    //如果线程池没有线程，那么直接执行任务，也就是说假设没有消费者，那么生产者直接消费产品.
    //而不把任务加入任务队列
  if (threads_.empty())
  {
    task();
  }
  //如果线程池有线程
  else
  {
      //加锁
    MutexLockGuard lock(mutex_);
        //加入任务队列
    queue_.push_back(task);
    //唤醒
    cond_.notify();
  }
}
//3----->>> 这是一个任务分配函数，线程池函数或者线程池里面的函数都可以到这里取出一个任务，然后在自己的线程中执行这和任务，返回一个任务指针
ThreadPool::Task ThreadPool::take()
{
  MutexLockGuard lock(mutex_);
  // always use a while-loop, due to spurious wakeup
  //防止假唤醒,running_ 用来接收线程池发送的“线程退出”命令
  while (queue_.empty() && running_)
  {
    cond_.wait();
  }
  Task task;
  if(!queue_.empty())
  {
    task = queue_.front();
    queue_.pop_front();
  }
  return task;
}
 
//2---->>线程池（可以把他当做主线程进行理解）里的线程的回调函数，可以在线程池函数中执行一些任务，并不是说只有线程池里面的函数才能执行任务，这能达到负载均衡的效果
void ThreadPool::runInThread()
{
  try
  {
        //测试线程池是否在运行 ，不运行马上停止线程池
    while (running_)
    {
      Task task(take()); //任务是函数，task == 函数指针
      if (task)
      {//执行任务，消费产品
        task();
      }
    }
  }
  catch (const Exception& ex)
  {
    fprintf(stderr, "exception caught in ThreadPool %s\n", name_.c_str());
    fprintf(stderr, "reason: %s\n", ex.what());
    fprintf(stderr, "stack trace: %s\n", ex.stackTrace());
    abort();
  }
  catch (const std::exception& ex)
  {
    fprintf(stderr, "exception caught in ThreadPool %s\n", name_.c_str());
    fprintf(stderr, "reason: %s\n", ex.what());
    abort();
  }
  catch (...)
  {
    fprintf(stderr, "unknown exception caught in ThreadPool %s\n", name_.c_str());
    throw; // rethrow
  }
}

```

### ThreadPool的测试程序

``` c++
#include <muduo/base/ThreadPool.h>
#include <muduo/base/CountDownLatch.h>
#include <muduo/base/CurrentThread.h>
 
#include <boost/bind.hpp>
#include <stdio.h>
 
void print()
{
  printf("tid=%d\n", muduo::CurrentThread::tid());
}
 
void printString(const std::string& str)
{
  printf("tid=%d, str=%s\n", muduo::CurrentThread::tid(), str.c_str());
}
 
int main()
{
  muduo::ThreadPool pool("MainThreadPool");
  pool.start(5);
 
  pool.run(print);
  pool.run(print);
  for (int i = 0; i < 100; ++i)
  {
    char buf[32];
    snprintf(buf, sizeof buf, "task %d", i);
    pool.run(boost::bind(printString, std::string(buf)));
  }
 //CountDowndLatch 作文任务放进去，子线会执行，然后唤醒pool，pool就会调用stop停止线程池
  muduo::CountDownLatch latch(1);
  pool.run(boost::bind(&muduo::CountDownLatch::countDown, &latch));
  latch.wait();
  pool.stop();
}

```

## [17] Singleton

### Singleton class uml

<center>![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/singleton.png)</center>


### Singleton源代码

``` c++
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
//
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#ifndef MUDUO_BASE_SINGLETON_H
#define MUDUO_BASE_SINGLETON_H
 
#include <boost/noncopyable.hpp>
#include <pthread.h>
#include <stdlib.h> // atexit
 
namespace muduo
{
 
template<typename T>
class Singleton : boost::noncopyable //不可复制的
{
 public:
  static T& instance() //单例一个对象
  {
    pthread_once(&ponce_, &Singleton::init); //线程安全，并且一次调用
    return *value_;
  }
 
 private:
  Singleton();
  ~Singleton();
 
    //初始化对象value
  static void init()
  {
    value_ = new T();
    //加入释放队列
    ::atexit(destroy);
  }
 
  static void destroy()
  {
      //T_must_be_complete_type 完全类型
    typedef char T_must_be_complete_type[sizeof(T) == 0 ? -1 : 1];
    //真正释放实例资源
    delete value_;
  }
 
 private:
  static pthread_once_t ponce_;
  static T*             value_;
};
 
template<typename T>
pthread_once_t Singleton<T>::ponce_ = PTHREAD_ONCE_INIT;
 
template<typename T>
T* Singleton<T>::value_ = NULL;
 
}
#endif
```

### 测试程序

``` c++
#include <muduo/base/Singleton.h>
#include <muduo/base/CurrentThread.h>
#include <muduo/base/Thread.h>
 
#include <boost/noncopyable.hpp>
#include <stdio.h>
 
class Test : boost::noncopyable
{
 public:
  Test()
  {
    printf("tid=%d, constructing %p\n", muduo::CurrentThread::tid(), this);
  }
 
  ~Test()
  {
    printf("tid=%d, destructing %p %s\n", muduo::CurrentThread::tid(), this, name_.c_str());
  }
 
  const muduo::string& name() const { return name_; }
  void setName(const muduo::string& n) { name_ = n; }
 
 private:
  muduo::string name_;
};
 
void threadFunc()
{
  printf("tid=%d, %p name=%s\n",
         muduo::CurrentThread::tid(),
         &muduo::Singleton<Test>::instance(),
         muduo::Singleton<Test>::instance().name().c_str());
  muduo::Singleton<Test>::instance().setName("only one, changed");
}
 
int main()
{
    //muduo::Singleton<Test>::instance() 单例化 Test
  muduo::Singleton<Test>::instance().setName("only one");
  muduo::Thread t1(threadFunc);
  t1.start();
  t1.join();
  printf("tid=%d, %p name=%s\n",
         muduo::CurrentThread::tid(),
         &muduo::Singleton<Test>::instance(),
         muduo::Singleton<Test>::instance().name().c_str());
}
```

### 程序输出

``` sh
[root@localhost bin]# ./singleton_test
tid=2777, constructing 0x87cc008
tid=2778, 0x87cc008 name=only one
tid=2777, 0x87cc008 name=only one, changed
tid=2777, destructing 0x87cc008 only one, changed
[root@localhost bin]#
```

## [18] ThreadLocal 

<center> ![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/ThreadLocalSingleton.png) </center>

### ThreadLocal 的源代码

``` c++
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
//
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#ifndef MUDUO_BASE_THREADLOCAL_H
#define MUDUO_BASE_THREADLOCAL_H
 
#include <boost/noncopyable.hpp>
#include <pthread.h>
 
namespace muduo
{
 
template<typename T>
class ThreadLocal : boost::noncopyable
{
 public:
     //构造函数，创建pkey
  ThreadLocal()
  {
     /*
     创建的键存放在pkey指定的内存单元，这个键可以被进程中的所有线程使用，但每个线程
把这个键与不同的线程 “私有数据地址” 进行关联。创建新键时，每个线程的数据地址设为NULL值。   
    ThreadLocal::destructor键的虚构函数   
     **/
    pthread_key_create(&pkey_, &ThreadLocal::destructor);
  }
 
  ~ThreadLocal()
  {
    pthread_key_delete(pkey_);
  }
    // 获取数据，如果数据还没有，则新建一个
  T& value()
  {
    T* perThreadValue = static_cast<T*>(pthread_getspecific(pkey_));
    if (!perThreadValue) {
      T* newObj = new T();
      //pthread_setspecific使pkey 和 私有数据进行关联
      pthread_setspecific(pkey_, newObj);
      perThreadValue = newObj;
    }
    return *perThreadValue;
  }
 
 private:
 
  static void destructor(void *x)
  {
    T* obj = static_cast<T*>(x);
    //判断是否为全类型，如果是则delete obj，
    typedef char T_must_be_complete_type[sizeof(T) == 0 ? -1 : 1];
    delete obj;
  }
 
 private:
  pthread_key_t pkey_;
};
 
}
#endif
```

### ThreadLocal 的测试程序

``` c++
#include <muduo/base/ThreadLocal.h>
#include <muduo/base/CurrentThread.h>
#include <muduo/base/Thread.h>
 
#include <boost/noncopyable.hpp>
#include <stdio.h>
 
class Test:boost::noncopyable
{
private :
    std::string name_;
public :
    Test(){
        printf("tid=%d,constructing %p\n",muduo::CurrentThread::tid(),this);
 
    }  
    ~Test(){
        printf("tid= %d,destructing %p %s\n",muduo::CurrentThread::tid(),this,name_.c_str());
    }
 
    const std::string &name()const{ return name_ ;}
    void setName(const std::string &n ){ name_ = n ;}
 
};
 
muduo::ThreadLocal<Test> testObj1;
muduo::ThreadLocal<Test> testObj2;
 
void print(){
    printf("tid =%d\tobj1 %p\tname=%s\n",muduo::CurrentThread::tid(),
        &testObj1.value(),
        testObj1.value().name().c_str());
    printf("tid=%d\tobj2 %p name=%s\n",
        muduo::CurrentThread::tid(),
        &testObj2.value(),
        testObj2.value().name().c_str());
}
 
void threadFunc(){
    print();
    testObj1.value().setName("changed 1") ;
    testObj2.value().setName("changed 2");
    print();
 
}
int main (void ){
    testObj1.value().setName("main One");
    print();
    muduo::Thread t1 (threadFunc) ;
    t1.start();
    t1.join();
    testObj2.value().setName("main two" ) ;
    print();
    pthread_exit(0);
    return  0 ;
 
}
```

### 程序输出

``` sh

[root@localhost bin]# ./threadlocal_test
tid=2915,constructing 0x9435028
tid =2915   obj1 0x9435028  name=main One
tid=2915,constructing 0x9435038
tid=2915    obj2 0x9435038 name=
tid=2916,constructing 0xb6c00468
tid =2916   obj1 0xb6c00468 name=
tid=2916,constructing 0xb6c00478
tid=2916    obj2 0xb6c00478 name=
tid =2916   obj1 0xb6c00468 name=changed 1
tid=2916    obj2 0xb6c00478 name=changed 2
tid= 2916,destructing 0xb6c00468 changed 1
tid= 2916,destructing 0xb6c00478 changed 2
tid =2915   obj1 0x9435028  name=main One
tid=2915    obj2 0x9435038 name=main two
tid= 2915,destructing 0x9435028 main One
tid= 2915,destructing 0x9435038 main two
[root@localhost bin]# ls

```

## [19] ThreadLocalSingle 

### ThreadLocalSingle class UML

<center>![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/ThreadLocalSingletonDeleter.png)</center>

### 源程序

``` c++
// Use of this source code is governed by a BSD-style license
// that can be found in the License file.
//
// Author: Shuo Chen (chenshuo at chenshuo dot com)
 
#ifndef MUDUO_BASE_THREADLOCALSINGLETON_H
#define MUDUO_BASE_THREADLOCALSINGLETON_H
 
#include <boost/noncopyable.hpp>
#include <assert.h>
#include <pthread.h>
/**
 
线程本地单例封装
---------------------------
-ponce_:pthread_once_t
-value:T*
----------------------
+instance():T&
<<create>>-Singleton()
<<destroy>>-Singleton()
-init():void
-destroy():void
---------------------------
***/
 
namespace muduo
{
 
//使用模板
template<typename T>
class ThreadLocalSingleton : boost::noncopyable //不可复制的
{
 public:
 
  static T& instance()
  {
    if (!t_value_)
    {
      t_value_ = new T();
      deleter_.set(t_value_);
    }
    return *t_value_;
  }
 
  static T* pointer()
  {
    return t_value_;
  }
 
 private:
 
  static void destructor(void* obj)
  {
    assert(obj == t_value_);
    //T_must_be_complete_type : 完全类型
    // Thread a;
    // a *p  ; //p这种类型就不是完全类型
    typedef char T_must_be_complete_type[sizeof(T) == 0 ? -1 : 1];
    //
    delete t_value_;
    t_value_ = 0;
  }
 
/**
Deleter 主要是用来使ThreadLocalSingleton::destructor作为pkey_的虚构回调函数
Deleter 只是虚构时只是pthread_key_delete(pkey_)而已
**/
  class Deleter
  {
   public:
    Deleter()
    {
      pthread_key_create(&pkey_, &ThreadLocalSingleton::destructor);
    }
 
    ~Deleter()
    {
      pthread_key_delete(pkey_);
    }
 
    void set(T* newObj)
    {
      assert(pthread_getspecific(pkey_) == NULL);
      pthread_setspecific(pkey_, newObj);
    }
 
    pthread_key_t pkey_;
  };
 
  static __thread T* t_value_; //只被执行一次，并且是线程安全 ====实际数据
                                // __thread 修饰的变量一定是全局的或者静态的
  static Deleter deleter_;      //消费实际数据函数
};
 
template<typename T>
__thread T* ThreadLocalSingleton<T>::t_value_ = 0;
 
template<typename T>
typename ThreadLocalSingleton<T>::Deleter ThreadLocalSingleton<T>::deleter_;
 
}
#endif
```

### 测试程序

``` erlang
#include <muduo/base/ThreadLocalSingleton.h>
#include <muduo/base/CurrentThread.h>
#include <muduo/base/Thread.h>
 
#include <boost/bind.hpp>
#include <boost/noncopyable.hpp>
#include <stdio.h>
 
class Test : boost::noncopyable
{
 public:
  Test()
  {
    printf("tid=%d, constructing %p\n", muduo::CurrentThread::tid(), this);
  }
 
  ~Test()
  {
    printf("tid=%d, destructing %p %s\n", muduo::CurrentThread::tid(), this, name_.c_str());
  }
 
  const std::string& name() const { return name_; }
  void setName(const std::string& n) { name_ = n; }
 
 private:
  std::string name_;
};
 
void threadFunc(const char* changeTo)
{
  printf("tid=%d, %p name=%s\n",
         muduo::CurrentThread::tid(),
         &muduo::ThreadLocalSingleton<Test>::instance(),
         muduo::ThreadLocalSingleton<Test>::instance().name().c_str());
  muduo::ThreadLocalSingleton<Test>::instance().setName(changeTo);
  printf("tid=%d, %p name=%s\n",
         muduo::CurrentThread::tid(),
         &muduo::ThreadLocalSingleton<Test>::instance(),
         muduo::ThreadLocalSingleton<Test>::instance().name().c_str());
 
  // no need to manually delete it
  // muduo::ThreadLocalSingleton<Test>::destroy();
}
 
int main()
{
  muduo::ThreadLocalSingleton<Test>::instance().setName("main one");
  muduo::Thread t1(boost::bind(threadFunc, "thread1"));
  muduo::Thread t2(boost::bind(threadFunc, "thread2"));
  t1.start();
  t2.start();
  t1.join();
  printf("tid=%d, %p name=%s\n",
         muduo::CurrentThread::tid(),
         &muduo::ThreadLocalSingleton<Test>::instance(),
         muduo::ThreadLocalSingleton<Test>::instance().name().c_str());
  t2.join();
 
  pthread_exit(0);
}
```

### 程序输出

``` c++
[root@localhost bin]# ./threadlocalsingleton_test
tid=2183, Obj address  0x89b4028, function Test
tid=2184, Obj address  0xb6c00468, function Test
tid=2184, obj address 0xb6c00468 name=
tid=2184, 0xb6c00468 name=thread1
tid=2184, destructing 0xb6c00468 thread1
tid=2183, 0x89b4028 name=main one
tid=2185, Obj address  0xb6c00468, function Test
tid=2185, obj address 0xb6c00468 name=
tid=2185, 0xb6c00468 name=thread2
tid=2185, destructing 0xb6c00468 thread2
tid=2183, destructing 0x89b4028 main one
[root@localhost bin]#
```

## [20] Logger-1

### 日志流程

<center> ![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/logger%281%29.png) </center>

- TRACE
指出比DEBUG粒度更细的一些信息事件（开发过程中使用）
- DEBUG
指出细粒度信息事件对调试应用程序是非常有帮助的。（开发过程中使用）
- INFO
表明消息在粗粒度级别上突出强调应用程序的运行过程。
- WARN
系统能正常运行，但可能会出现潜在错误的情形。
- ERROR
指出虽然发生错误事件，但仍然不影响系统的继续运行。
- FATAL
指出每个严重的错误事件将会导致应用程序的退出。


### 测试程序1

``` erlang
#include <muduo/base/Logging.h>
#include <errno.h>
 
using namespace muduo;
 
int main()
{
 
 // Logger.LogLevel = LogLevel::INFO ;
//  muduo::Logger.setLogLevel( LogLevel::INFO ) ;
    LOG_TRACE<<"trace ...";
    LOG_DEBUG<<"debug ...";
    LOG_INFO<<"info ...";
    LOG_WARN<<"warn ...";
    LOG_ERROR<<"error ...";
    //LOG_FATAL<<"fatal ...";
    errno = 13;
    LOG_SYSERR<<"syserr ...";
    LOG_SYSFATAL<<"sysfatal ...";
    return 0;
}
```

### 程序输出

``` erlang
[root@localhost bin]# ./log_test1
20131016 15:34:26.231014Z  3904 TRACE main trace ... - Log_test1.cc:11
20131016 15:34:26.231251Z  3904 DEBUG main debug ... - Log_test1.cc:12
20131016 15:34:26.231264Z  3904 INFO  info ... - Log_test1.cc:13
20131016 15:34:26.231274Z  3904 WARN  warn ... - Log_test1.cc:14
20131016 15:34:26.231285Z  3904 ERROR error ... - Log_test1.cc:15
20131016 15:34:26.231293Z  3904 ERROR Permission denied (errno=13) syserr ... - Log_test1.cc:18
20131016 15:34:26.231317Z  3904 FATAL Permission denied (errno=13) sysfatal ... - Log_test1.cc:19
Aborted
[root@localhost bin]#
```

### 测试程序2

``` erlang
#include <muduo/base/LogFile.h>
#include <muduo/base/Logging.h>
 
boost::scoped_ptr<muduo::LogFile> g_logFile;
 
void outputFunc(const char* msg, int len)
{
  g_logFile->append(msg, len);
}
 
void flushFunc()
{
  g_logFile->flush();
}
 
int main(int argc, char* argv[])
{
  char name[256];
  strncpy(name, argv[0], 256);
  g_logFile.reset(new muduo::LogFile(::basename(name), 200*1000));
  muduo::Logger::setOutput(outputFunc);
  muduo::Logger::setFlush(flushFunc);
 
  muduo::string line = "1234567890 abcdefghijklmnopqrstuvwxyz ABCDEFGHIJKLMNOPQRSTUVWXYZ ";
 
  for (int i = 0; i < 10000; ++i)
  {
    LOG_INFO << line << i;
 
    usleep(1000);
  }
}
```

### 程序输出

``` erlang
[root@localhost bin]# cat /tmp/muduo_log
20131016 15:50:46.099054Z  4358 TRACE main trace ... - Log_test2.cc:28
20131016 15:50:46.099182Z  4358 DEBUG main debug ... - Log_test2.cc:29
20131016 15:50:46.099191Z  4358 INFO  info ... - Log_test2.cc:30
20131016 15:50:46.099197Z  4358 WARN  warn ... - Log_test2.cc:31
20131016 15:50:46.099201Z  4358 ERROR error ... - Log_test2.cc:32
20131016 15:50:46.099206Z  4358 ERROR Permission denied (errno=13) syserr ... - Log_test2.cc:35
[root@localhost bin]#
```

### 测试程序3

``` erlang
#include <muduo/base/Logging.h>
#include <muduo/base/LogFile.h>
#include <muduo/base/ThreadPool.h>
 
#include <stdio.h>
 
int g_total;
FILE* g_file;
boost::scoped_ptr<muduo::LogFile> g_logFile;
 
void dummyOutput(const char* msg, int len)
{
  g_total += len;
  if (g_file)
  {
    fwrite(msg, 1, len, g_file);
  }
  else if (g_logFile)
  {
    g_logFile->append(msg, len);
  }
}
 
void bench(const char* type)
{
  muduo::Logger::setOutput(dummyOutput);
  muduo::Timestamp start(muduo::Timestamp::now());
  g_total = 0;
 
  int n = 1000*1000;
  const bool kLongLog = false;
  muduo::string empty = " ";
  muduo::string longStr(3000, 'X');
  longStr += " ";
  for (int i = 0; i < n; ++i)
  {
    LOG_INFO << "Hello 0123456789" << " abcdefghijklmnopqrstuvwxyz"
             << (kLongLog ? longStr : empty)
             << i;
  }
  muduo::Timestamp end(muduo::Timestamp::now());
  double seconds = timeDifference(end, start);
  printf("%12s: %f seconds, %d bytes, %10.2f msg/s, %.2f MiB/s\n",
         type, seconds, g_total, n / seconds, g_total / seconds / (1024 * 1024));
}
 
void logInThread()
{
  LOG_INFO << "logInThread";
  usleep(1000);
}
 
int main()
{
  getppid(); // for ltrace and strace
    //线程池 ， 名字"pool"
  muduo::ThreadPool pool("pool");
    //五个线程
  pool.start(5);
  //五个任务
  pool.run(logInThread);
  pool.run(logInThread);
  pool.run(logInThread);
  pool.run(logInThread);
  pool.run(logInThread);
 
  LOG_TRACE << "trace";
  LOG_DEBUG << "debug";
  LOG_INFO << "Hello";
  LOG_WARN << "World";
  LOG_ERROR << "Error";
  LOG_INFO << sizeof(muduo::Logger);
  LOG_INFO << sizeof(muduo::LogStream);
  LOG_INFO << sizeof(muduo::Fmt);
  LOG_INFO << sizeof(muduo::LogStream::Buffer);
 
  sleep(1);
  //
  bench("nop");
 
  char buffer[64*1024];
 
  g_file = fopen("/dev/null", "w");
  setbuffer(g_file, buffer, sizeof buffer);
  bench("/dev/null");
  fclose(g_file);
 
  g_file = fopen("/tmp/log", "w");
  setbuffer(g_file, buffer, sizeof buffer);
  bench("/tmp/log");
  fclose(g_file);
 
  g_file = NULL;
  //线程安全的方式
  g_logFile.reset(new muduo::LogFile("test_log_st", 500*1000*1000, false));
  bench("test_log_st");
    //线程不安全的方式
  g_logFile.reset(new muduo::LogFile("test_log_mt", 500*1000*1000, true));
  bench("test_log_mt");
  g_logFile.reset();
}
```

## [21] Logger-2 

<center> ![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/logger2.png)</center>

### LogStream类图

<center> ![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/logstream.png)</center>

### FixedBuffer类图

<center>![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/fixedbuffer.png)</center>

### Logger类图

<center>![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/logger3.png)</center>

### Impl类图

<center>![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/loggerimpl.png)</center>

### LogStream的测试程序1  

logstream_test.cc

``` c++
#include <muduo/base/LogStream.h>
 
#include <limits>
#include <stdint.h>
 
//#define BOOST_TEST_MODULE LogStreamTest
#define BOOST_TEST_MAIN
#define BOOST_TEST_DYN_LINK
#include <boost/test/unit_test.hpp>
 
using muduo::string;
 
BOOST_AUTO_TEST_CASE(testLogStreamBooleans)
{
  muduo::LogStream os;
  const muduo::LogStream::Buffer& buf = os.buffer();
  BOOST_CHECK_EQUAL(buf.asString(), string(""));
  os << true;
  BOOST_CHECK_EQUAL(buf.asString(), string("1"));
  os << '\n';
  BOOST_CHECK_EQUAL(buf.asString(), string("1\n"));
  os << false;
  BOOST_CHECK_EQUAL(buf.asString(), string("1\n0"));
}
 
BOOST_AUTO_TEST_CASE(testLogStreamIntegers)
{
  muduo::LogStream os;
  const muduo::LogStream::Buffer& buf = os.buffer();
  BOOST_CHECK_EQUAL(buf.asString(), string(""));
  os << 1;
  BOOST_CHECK_EQUAL(buf.asString(), string("1"));
  os << 0;
  BOOST_CHECK_EQUAL(buf.asString(), string("10"));
  os << -1;
  BOOST_CHECK_EQUAL(buf.asString(), string("10-1"));
  os.resetBuffer();
 
  os << 0 << " " << 123 << 'x' << 0x64;
  BOOST_CHECK_EQUAL(buf.asString(), string("0 123x100"));
}
 
BOOST_AUTO_TEST_CASE(testLogStreamIntegerLimits)
{
  muduo::LogStream os;
  const muduo::LogStream::Buffer& buf = os.buffer();
  os << -2147483647;
  BOOST_CHECK_EQUAL(buf.asString(), string("-2147483647"));
  os << static_cast<int>(-2147483647 - 1);
  BOOST_CHECK_EQUAL(buf.asString(), string("-2147483647-2147483648"));
  os << ' ';
  os << 2147483647;
  BOOST_CHECK_EQUAL(buf.asString(), string("-2147483647-2147483648 2147483647"));
  os.resetBuffer();
 
  os << std::numeric_limits<int16_t>::min();
  BOOST_CHECK_EQUAL(buf.asString(), string("-32768"));
  os.resetBuffer();
 
  os << std::numeric_limits<int16_t>::max();
  BOOST_CHECK_EQUAL(buf.asString(), string("32767"));
  os.resetBuffer();
 
  os << std::numeric_limits<uint16_t>::min();
  BOOST_CHECK_EQUAL(buf.asString(), string("0"));
  os.resetBuffer();
 
  os << std::numeric_limits<uint16_t>::max();
  BOOST_CHECK_EQUAL(buf.asString(), string("65535"));
  os.resetBuffer();
 
  os << std::numeric_limits<int32_t>::min();
  BOOST_CHECK_EQUAL(buf.asString(), string("-2147483648"));
  os.resetBuffer();
 
  os << std::numeric_limits<int32_t>::max();
  BOOST_CHECK_EQUAL(buf.asString(), string("2147483647"));
  os.resetBuffer();
 
  os << std::numeric_limits<uint32_t>::min();
  BOOST_CHECK_EQUAL(buf.asString(), string("0"));
  os.resetBuffer();
 
  os << std::numeric_limits<uint32_t>::max();
  BOOST_CHECK_EQUAL(buf.asString(), string("4294967295"));
  os.resetBuffer();
 
  os << std::numeric_limits<int64_t>::min();
  BOOST_CHECK_EQUAL(buf.asString(), string("-9223372036854775808"));
  os.resetBuffer();
 
  os << std::numeric_limits<int64_t>::max();
  BOOST_CHECK_EQUAL(buf.asString(), string("9223372036854775807"));
  os.resetBuffer();
 
  os << std::numeric_limits<uint64_t>::min();
  BOOST_CHECK_EQUAL(buf.asString(), string("0"));
  os.resetBuffer();
 
  os << std::numeric_limits<uint64_t>::max();
  BOOST_CHECK_EQUAL(buf.asString(), string("18446744073709551615"));
  os.resetBuffer();
 
  int16_t a = 0;
  int32_t b = 0;
  int64_t c = 0;
  os << a;
  os << b;
  os << c;
  BOOST_CHECK_EQUAL(buf.asString(), string("000"));
}
 
BOOST_AUTO_TEST_CASE(testLogStreamFloats)
{
  muduo::LogStream os;
  const muduo::LogStream::Buffer& buf = os.buffer();
 
  os << 0.0;
  BOOST_CHECK_EQUAL(buf.asString(), string("0"));
  os.resetBuffer();
 
  os << 1.0;
  BOOST_CHECK_EQUAL(buf.asString(), string("1"));
  os.resetBuffer();
 
  os << 0.1;
  BOOST_CHECK_EQUAL(buf.asString(), string("0.1"));
  os.resetBuffer();
 
  os << 0.05;
  BOOST_CHECK_EQUAL(buf.asString(), string("0.05"));
  os.resetBuffer();
 
  os << 0.15;
  BOOST_CHECK_EQUAL(buf.asString(), string("0.15"));
  os.resetBuffer();
 
  double a = 0.1;
  os << a;
  BOOST_CHECK_EQUAL(buf.asString(), string("0.1"));
  os.resetBuffer();
 
  double b = 0.05;
  os << b;
  BOOST_CHECK_EQUAL(buf.asString(), string("0.05"));
  os.resetBuffer();
 
  double c = 0.15;
  os << c;
  BOOST_CHECK_EQUAL(buf.asString(), string("0.15"));
  os.resetBuffer();
 
  os << a+b;
  BOOST_CHECK_EQUAL(buf.asString(), string("0.15"));
  os.resetBuffer();
 
  BOOST_CHECK(a+b != c);
 
  os << 1.23456789;
  BOOST_CHECK_EQUAL(buf.asString(), string("1.23456789"));
  os.resetBuffer();
 
  os << 1.234567;
  BOOST_CHECK_EQUAL(buf.asString(), string("1.234567"));
  os.resetBuffer();
 
  os << -123.456;
  BOOST_CHECK_EQUAL(buf.asString(), string("-123.456"));
  os.resetBuffer();
}
 
BOOST_AUTO_TEST_CASE(testLogStreamVoid)
{
  muduo::LogStream os;
  const muduo::LogStream::Buffer& buf = os.buffer();
 
  os << static_cast<void*>(0);
  BOOST_CHECK_EQUAL(buf.asString(), string("0x0"));
  os.resetBuffer();
 
  os << reinterpret_cast<void*>(8888);
  BOOST_CHECK_EQUAL(buf.asString(), string("0x22B8"));
  os.resetBuffer();
}
 
BOOST_AUTO_TEST_CASE(testLogStreamStrings)
{
  muduo::LogStream os;
  const muduo::LogStream::Buffer& buf = os.buffer();
 
  os << "Hello ";
  BOOST_CHECK_EQUAL(buf.asString(), string("Hello "));
 
  string chenshuo = "Shuo Chen";
  os << chenshuo;
  BOOST_CHECK_EQUAL(buf.asString(), string("Hello Shuo Chen"));
}
 
BOOST_AUTO_TEST_CASE(testLogStreamFmts)
{
  muduo::LogStream os;
  const muduo::LogStream::Buffer& buf = os.buffer();
 
  os << muduo::Fmt("%4d", 1);
  BOOST_CHECK_EQUAL(buf.asString(), string("   1"));
  os.resetBuffer();
 
  os << muduo::Fmt("%4.2f", 1.2);
  BOOST_CHECK_EQUAL(buf.asString(), string("1.20"));
  os.resetBuffer();
 
  os << muduo::Fmt("%4.2f", 1.2) << muduo::Fmt("%4d", 43);
  BOOST_CHECK_EQUAL(buf.asString(), string("1.20  43"));
  os.resetBuffer();
}
 
BOOST_AUTO_TEST_CASE(testLogStreamLong)
{
  muduo::LogStream os;
  const muduo::LogStream::Buffer& buf = os.buffer();
  for (int i = 0; i < 399; ++i)
  {
    os << "123456789 ";
    BOOST_CHECK_EQUAL(buf.length(), 10*(i+1));
    BOOST_CHECK_EQUAL(buf.avail(), 4000 - 10*(i+1));
  }
 
  os << "abcdefghi ";
  BOOST_CHECK_EQUAL(buf.length(), 3990);
  BOOST_CHECK_EQUAL(buf.avail(), 10);
 
  os << "abcdefghi";
  BOOST_CHECK_EQUAL(buf.length(), 3999);
  BOOST_CHECK_EQUAL(buf.avail(), 1);
}
```

Logstream 测试程序2 logstream_bench.cc 性能测试，测试不停数据类型的效率

``` c++
#include <muduo/base/LogStream.h>
#include <muduo/base/Timestamp.h>
 
#include <sstream>
#include <stdio.h>
#define __STDC_FORMAT_MACROS
#include <inttypes.h>
 
using namespace muduo;
 
const size_t N = 1000000;
 
#pragma GCC diagnostic ignored "-Wold-style-cast"
 
template<typename T>
void benchPrintf(const char* fmt)
{
  char buf[32];
  Timestamp start(Timestamp::now());
  for (size_t i = 0; i < N; ++i)
    snprintf(buf, sizeof buf, fmt, (T)(i));
  Timestamp end(Timestamp::now());
 
  printf("benchPrintf %f\n", timeDifference(end, start));
}
 
template<typename T>
void benchStringStream()
{
  Timestamp start(Timestamp::now());
  std::ostringstream os;
 
  for (size_t i = 0; i < N; ++i)
  {
    os << (T)(i);
    os.seekp(0, std::ios_base::beg);
  }
  Timestamp end(Timestamp::now());
 
  printf("benchStringStream %f\n", timeDifference(end, start));
}
 
template<typename T>
void benchLogStream()
{
  Timestamp start(Timestamp::now());
  LogStream os;
  for (size_t i = 0; i < N; ++i)
  {
    os << (T)(i);
    os.resetBuffer();
  }
  Timestamp end(Timestamp::now());
 
  printf("benchLogStream %f\n", timeDifference(end, start));
}
 
int main()
{
  benchPrintf<int>("%d");
 
  puts("int");
  benchPrintf<int>("%d");
  benchStringStream<int>();
  benchLogStream<int>();
 
  puts("double");
  benchPrintf<double>("%.12g");
  benchStringStream<double>();
  benchLogStream<double>();
 
  puts("int64_t");
  benchPrintf<int64_t>("%" PRId64);
  benchStringStream<int64_t>();
  benchLogStream<int64_t>();
 
  puts("void*");
  benchPrintf<void*>("%p");
  benchStringStream<void*>();
  benchLogStream<void*>();
 
}
```

## [21] traits 技术

``` c++
//
// Redistribution and use in source and binary forms, with or without
// modification, are permitted provided that the following conditions are
// met:
//
//     * Redistributions of source code must retain the above copyright
// notice, this list of conditions and the following disclaimer.
//     * Redistributions in binary form must reproduce the above
// copyright notice, this list of conditions and the following disclaimer
// in the documentation and/or other materials provided with the
// distribution.
//     * Neither the name of Google Inc. nor the names of its
// contributors may be used to endorse or promote products derived from
// this software without specific prior written permission.
//
// THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
// "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
// LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
// A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
// OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
// SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
// LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
// DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
// THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
// (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
// OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
//
// Author: Sanjay Ghemawat
//
// A string like object that points into another piece of memory.
// Useful for providing an interface that allows clients to easily
// pass in either a "const char*" or a "string".
//
// Arghh!  I wish C++ literals were automatically of type "string".
 
#ifndef MUDUO_BASE_STRINGPIECE_H
#define MUDUO_BASE_STRINGPIECE_H
 
#include <string.h>
#include <iosfwd>    // for ostream forward-declaration
 
#include <muduo/base/Types.h>
#ifndef MUDUO_STD_STRING
#include <string>
#endif
 
namespace muduo {
 
class StringPiece {
 private:
  const char*   ptr_;
  int           length_;
 
 public:
  // We provide non-explicit singleton constructors so users can pass
  // in a "const char*" or a "string" wherever a "StringPiece" is
  // expected.
  StringPiece()
    : ptr_(NULL), length_(0) { }
  StringPiece(const char* str)
    : ptr_(str), length_(static_cast<int>(strlen(ptr_))) { }
  StringPiece(const unsigned char* str)
    : ptr_(reinterpret_cast<const char*>(str)),
      length_(static_cast<int>(strlen(ptr_))) { }
  StringPiece(const string& str)
    : ptr_(str.data()), length_(static_cast<int>(str.size())) { }
#ifndef MUDUO_STD_STRING
  StringPiece(const std::string& str)
    : ptr_(str.data()), length_(static_cast<int>(str.size())) { }
#endif
  StringPiece(const char* offset, int len)
    : ptr_(offset), length_(len) { }
 
  // data() may return a pointer to a buffer with embedded NULs, and the
  // returned buffer may or may not be null terminated.  Therefore it is
  // typically a mistake to pass data() to a routine that expects a NUL
  // terminated string.  Use "as_string().c_str()" if you really need to do
  // this.  Or better yet, change your routine so it does not rely on NUL
  // termination.
  const char* data() const { return ptr_; }
  int size() const { return length_; }
  bool empty() const { return length_ == 0; }
 
  void clear() { ptr_ = NULL; length_ = 0; }
  void set(const char* buffer, int len) { ptr_ = buffer; length_ = len; }
  void set(const char* str) {
    ptr_ = str;
    length_ = static_cast<int>(strlen(str));
  }
  void set(const void* buffer, int len) {
    ptr_ = reinterpret_cast<const char*>(buffer);
    length_ = len;
  }
 
  char operator[](int i) const { return ptr_[i]; }
 
  void remove_prefix(int n) {
    ptr_ += n;
    length_ -= n;
  }
 
  void remove_suffix(int n) {
    length_ -= n;
  }
 
  bool operator==(const StringPiece& x) const {
    return ((length_ == x.length_) &&
            (memcmp(ptr_, x.ptr_, length_) == 0));
  }
  bool operator!=(const StringPiece& x) const {
    return !(*this == x);
  }
 
#define STRINGPIECE_BINARY_PREDICATE(cmp,auxcmp)                             \
  bool operator cmp (const StringPiece& x) const {                           \
    int r = memcmp(ptr_, x.ptr_, length_ < x.length_ ? length_ : x.length_); \
    return ((r auxcmp 0) || ((r == 0) && (length_ cmp x.length_)));          \
  }
  STRINGPIECE_BINARY_PREDICATE(<,  <);
  STRINGPIECE_BINARY_PREDICATE(<=, <);
  STRINGPIECE_BINARY_PREDICATE(>=, >);
  STRINGPIECE_BINARY_PREDICATE(>,  >);
#undef STRINGPIECE_BINARY_PREDICATE
 
  int compare(const StringPiece& x) const {
    int r = memcmp(ptr_, x.ptr_, length_ < x.length_ ? length_ : x.length_);
    if (r == 0) {
      if (length_ < x.length_) r = -1;
      else if (length_ > x.length_) r = +1;
    }
    return r;
  }
 
  string as_string() const {
    return string(data(), size());
  }
 
  void CopyToString(string* target) const {
    target->assign(ptr_, length_);
  }
 
#ifndef MUDUO_STD_STRING
  void CopyToStdString(std::string* target) const {
    target->assign(ptr_, length_);
  }
#endif
 
  // Does "this" start with "x"
  bool starts_with(const StringPiece& x) const {
    return ((length_ >= x.length_) && (memcmp(ptr_, x.ptr_, x.length_) == 0));
  }
};
 
}   // namespace muduo
 
// ------------------------------------------------------------------
// Functions used to create STL containers that use StringPiece
//  Remember that a StringPiece's lifetime had better be less than
//  that of the underlying string or char*.  If it is not, then you
//  cannot safely store a StringPiece into an STL container
// ------------------------------------------------------------------
 
#ifdef HAVE_TYPE_TRAITS
// This makes vector<StringPiece> really fast for some STL implementations
template<> struct __type_traits<muduo::StringPiece> {
  typedef __true_type    has_trivial_default_constructor;
  typedef __true_type    has_trivial_copy_constructor;
  typedef __true_type    has_trivial_assignment_operator;
  typedef __true_type    has_trivial_destructor;
  typedef __true_type    is_POD_type;
};
#endif
 
// allow StringPiece to be logged
std::ostream& operator<<(std::ostream& o, const muduo::StringPiece& piece);
 
#endif  // MUDUO_BASE_STRINGPIECE_H

```

## [22] LogFile

日志滚动

        日志滚动条件
        1、文件大小（例如每写满1G换下一个文件）
        2、时间（每天零点新建一个日志文件，不论前一个文件是否写满）
        一个典型的日志文件名
        logfile_test.20130411-115604.popo.7743.log

### Logger 类图


<center>![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/loggerclass.png)</center>

### File类图

<center>![](http://laohanlinux.github.io/images/img/blog/muduo_base_library/fileclass.png)</center>

file类是LogFile的一个内置类


### LogFile头文件(logfile.h)

``` c++
#ifndef MUDUO_BASE_LOGFILE_H
#define MUDUO_BASE_LOGFILE_H
 
#include <muduo/base/Mutex.h>
#include <muduo/base/Types.h>
 
#include <boost/noncopyable.hpp>
#include <boost/scoped_ptr.hpp>
 
namespace muduo
{
 
class LogFile : boost::noncopyable
{
 public:
  LogFile(const string& basename,
          size_t rollSize,
          bool threadSafe = true,
          int flushInterval = 3);
  ~LogFile();
 
  void append(const char* logline, int len);
  void flush();
 
 private:
  void append_unlocked(const char* logline, int len);
 
  static string getLogFileName(const string& basename, time_t* now); //获取日志文件路径
  void rollFile(); //滚动日志
 
  const string basename_; //日志文件basename
  const size_t rollSize_; //日志文件达到 rolSize_ 换一个新文件
  const int flushInterval_; //日志文件写入间隔时间 ，不是每一次的日志都直接写入文件，这样也有助于提高效率
 
  int count_; //计数器 ，当达到kCheckTimeRoll_时，会去检测是否要滚动日志文件
 
  boost::scoped_ptr<MutexLock> mutex_;
  time_t startOfPeriod_; //开始记录日志时间(调整至零点)
  time_t lastRoll_; //上一次滚动日志文件时间
  time_t lastFlush_;//上一次日志写入文件时间
  class File;               //日志文件类
  boost::scoped_ptr<File> file_; //日子文件指针，这是一个智能指针
 
  const static int kCheckTimeRoll_ = 1024; //
  const static int kRollPerSeconds_ = 60*60*24; //一天
};
 
}
#endif  // MUDUO_BASE_LOGFILE_H
```

### LogFile的源文件

``` c++
#include <muduo/base/LogFile.h>
#include <muduo/base/Logging.h> // strerror_tl
#include <muduo/base/ProcessInfo.h>
 
#include <assert.h>
#include <stdio.h>
#include <time.h>
 
using namespace muduo;
 
// not thread safe 不是线程安全
class LogFile::File : boost::noncopyable //不可复制
{
 public:、
    /********
    //File 类
 
    **********/
 
  explicit File(const string& filename)
    : fp_(::fopen(filename.data(), "ae")),
      writtenBytes_(0)
  {
    assert(fp_);  //断言fp_指针是否为空
    ::setbuffer(fp_, buffer_, sizeof buffer_); //是指缓冲区的大小
    // posix_fadvise POSIX_FADV_DONTNEED ?
  }
 
    //虚构函数 ， 关闭fp_指针
  ~File()
  {
    ::fclose(fp_);
  }
//追加 日志
  void append(const char* logline, const size_t len)
  {
    size_t n = write(logline, len);
    size_t remain = len - n;
    while (remain > 0)
    {
      size_t x = write(logline + n, remain);
      if (x == 0)
      {
        int err = ferror(fp_);
        if (err)
        {
          fprintf(stderr, "LogFile::File::append() failed %s\n", strerror_tl(err));
        }
        break;
      }
      n += x;
      remain = len - n; // remain -= x
    }
 
    writtenBytes_ += len;
  }
 
//清空缓冲区
  void flush()
  {
    ::fflush(fp_);
  }
 
  size_t writtenBytes() const { return writtenBytes_; }
 
 private:
//不加锁写入文件，logfile已经加锁了
  size_t write(const char* logline, size_t len)
  {
#undef fwrite_unlocked
    return ::fwrite_unlocked(logline, 1, len, fp_);
  }
 
  FILE* fp_;
  char buffer_[64*1024];
  size_t writtenBytes_;
};
 
/*******
 
LogFile类
*******/
LogFile::LogFile(const string& basename,
                 size_t rollSize,      
                 bool threadSafe,          
                 int flushInterval) 
  : basename_(basename),//文件basename
    rollSize_(rollSize),//日志滚动大小界限
    flushInterval_(flushInterval), //清空缓冲区
    count_(0),
    mutex_(threadSafe ? new MutexLock : NULL),//是否以线程安全的方式进行操作
    startOfPeriod_(0),
    lastRoll_(0),
    lastFlush_(0)
{
    //断言 ，basename 是否包含"/"
  assert(basename.find('/') == string::npos);
    //滚动日志
  rollFile();
}
 
LogFile::~LogFile()
{
}
 
//追加 日志记录
void LogFile::append(const char* logline, int len)
{
  if (mutex_)
  {
    MutexLockGuard lock(*mutex_);
    append_unlocked(logline, len);
  }
  else
  {
    append_unlocked(logline, len);
  }
}
 
void LogFile::flush()
{
  if (mutex_)
  {
    MutexLockGuard lock(*mutex_);
    file_->flush();
  }
  else
  {
    file_->flush();
  }
}
 
void LogFile::append_unlocked(const char* logline, int len)
{
  file_->append(logline, len);
    //判断是否要滚动日志文件了 ，大小上测试
  if (file_->writtenBytes() > rollSize_)
  {
    //滚动日志文件
    rollFile();
  }
  else
  {
    //时间上测试 ，就是计数器
    if (count_ > kCheckTimeRoll_)
    {
      count_ = 0;
      time_t now = ::time(NULL);
      time_t thisPeriod_ = now / kRollPerSeconds_ * kRollPerSeconds_;
      if (thisPeriod_ != startOfPeriod_)
      {
        rollFile();
      }
      //是否应该清空数据
      else if (now - lastFlush_ > flushInterval_)
      {
        lastFlush_ = now;
        file_->flush();
      }
    }
    //计数器++
    else
    {
      ++count_;
    }
  }
}
 
//日志滚动
void LogFile::rollFile()
{
  time_t now = 0;
  string filename = getLogFileName(basename_, &now);
  time_t start = now / kRollPerSeconds_ * kRollPerSeconds_;
//如果是时间上的滚动，那么就要更新一下时间了
  if (now > lastRoll_)
  {
    lastRoll_ = now;
    lastFlush_ = now;
    startOfPeriod_ = start;
    file_.reset(new File(filename));
  }
}
 
//得到日志文件名
string LogFile::getLogFileName(const string& basename, time_t* now)
{
  string filename;
  filename.reserve(basename.size() + 64);
  filename = basename;
 
  char timebuf[32];
  char pidbuf[32];
  struct tm tm;
  *now = time(NULL);
  gmtime_r(now, &tm); // FIXME: localtime_r ?
  strftime(timebuf, sizeof timebuf, ".%Y%m%d-%H%M%S.", &tm);
  filename += timebuf;
  filename += ProcessInfo::hostname();
  snprintf(pidbuf, sizeof pidbuf, ".%d", ProcessInfo::pid());
  filename += pidbuf;
  filename += ".log";
 
  return filename;
}

```

### LogFile测试类 LogFile_Test.cc

``` c++
#include <muduo/base/LogFile.h>
#include <muduo/base/Logging.h>
 
boost::scoped_ptr<muduo::LogFile> g_logFile;
 
//msg日志内容，日志长度
void outputFunc(const char* msg, int len)
{
  g_logFile->append(msg, len);
}
 
void flushFunc()
{
  g_logFile->flush();
}
 
int main(int argc, char* argv[])
{
  char name[256];
 
  strncpy(name, argv[0], 256);
  //日志滚动的大小200*1000 = 196k
  g_logFile.reset(new muduo::LogFile(::basename(name), 200*1000));
 
    //Logger的输出函数
  muduo::Logger::setOutput(outputFunc);
    //Logger的缓冲区清空函数
  muduo::Logger::setFlush(flushFunc);
        //日志
  muduo::string line = "1234567890 abcdefghijklmnopqrstuvwxyz ABCDEFGHIJKLMNOPQRSTUVWXYZ ";
    //10000条日志记录
  for (int i = 0; i < 10000; ++i)
  {
    LOG_INFO << line << i;
 
    usleep(1000);
  }
}
```

### 程序输出

``` sh
[root@localhost bin]# ls -lh logfile_test*
-rwxr-xr-x. 1 root root 610K Oct 17 16:02 logfile_test
-rw-r--r--. 1 root root 196K Oct 17 16:03 logfile_test.20131017-080304.localhost.localdomain.2656.log
-rw-r--r--. 1 root root 196K Oct 17 16:03 logfile_test.20131017-080309.localhost.localdomain.2656.log
-rw-r--r--. 1 root root 196K Oct 17 16:03 logfile_test.20131017-080312.localhost.localdomain.2656.log
-rw-r--r--. 1 root root 196K Oct 17 16:03 logfile_test.20131017-080314.localhost.localdomain.2656.log
-rw-r--r--. 1 root root 196K Oct 17 16:03 logfile_test.20131017-080316.localhost.localdomain.2656.log
-rw-r--r--. 1 root root 196K Oct 17 16:03 logfile_test.20131017-080319.localhost.localdomain.2656.log
-rw-r--r--. 1 root root  87K Oct 17 16:03 logfile_test.20131017-080321.localhost.localdomain.2656.log
[root@localhost bin]#
```
