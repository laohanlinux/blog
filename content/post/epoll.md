---
title: Linux epoll
date: 2016-03-20 11:54:32
tags:
- epoll
- timerfd
categories: 
- Linux Program
description: "epoll的简单使用,同时结合了timerfd"
---

# epoll 

`epoll`是一种事件驱动的`io`模式，`epoll`不仅可以应用在网络`io`上，时钟也是适用，得益于`linux/Unix`的一切皆为文件理论。这也意味着系统的所有资源都可以用`文件`来表示，这些资源都用一个具有`read()`,`write()`, `close()`接口的文件描述符进行管理。

`Linux`也提供了文件描述符的`监听`事件功能，比如`在一个目录下创建一个文件`和`时钟超时`的事件。


`epoll`的触发模式有2种：1、水平触发（电平触发-LT）；2、边沿触发（ET）

> EPOLL 在有些情况下是不建议使用的
如：已连接套接字不多，并且这些套接字非常活跃;都是活跃，那就不用等什么事件了，直接操作就完事了。

- 水平触发

`Write EPOLLOUT EVENT`:
高电平: `writebuf`内核有空闲空间，我们就说他处于高电平状态，也就是一直处于活跃状态。如果应用没有及时写入数据，此时可能会产生`busy waitting loop`
低电平: 当`writebuf`内核没有空闲空间，我们就说他处于低电平状态，没有激活。

或者这样解析：
内核中的socket接收缓冲区为空 =》 低电平
内核中的socket接收缓冲区不为空 =》 高电平

`Read EPOLLIN EVENT`:

高电平：`ReadBuf`内核有数据可读，如果应用没有及时把这些数据读取掉，那么也可能产生`busy waitting loop`的情况
低电平：`ReadBuf`内核没有数据可读，处于低电平状态，没有被激活。

或者这样解析：
内核中的socket发送缓冲区不满 =》 高电平
内核中的socket发送缓冲区满 =》 低电平

总之，应用在使用`epoll`是尽量写满内核`写缓冲区`和读空内核`读缓存区`,避免空闲等待.

- 边缘触发

在该模式下，一开始我们就要关注`EPOLLIN`事件和`EPOLLOU`T事件 ， 此时`writbuf`一直处于高电平，不会触发`EPOLLOUT`事件.而`EPOLLIN`可能会产生，因为开始`Readbuf`是空的，如果在 `epoll_wait`前，`readbuf`有数据了，那么就有`unreadablr--->readable`,
也就是产生了`EPOLLLIN`事件，这也是为什么监听`socket` 能够接收到外来请求的原因。
关注`EPOLLIN` 和 `EPOLLOUT`事件后，我们也没必要取消他们的关注，只有到断开他们的`socketfd`时 ， 我们才需要取消。

区别：LT事件不会丢弃，而是只要读buffer里面有数据可以让用户读取，则不断的通知你。而ET则只在事件发生之时通知。

## timerfd 

[link: timerfd-source](http://lxr.free-electrons.com/source/fs/timerfd.c)

一般文件描述的使用步骤都是调用一个初始化函数来创建一个文件描述符，也就是句柄。如：

```
#include<sys/timerfd.h>

int fd = timerfd_create(CLOCK_MONOTONIC, TFD_MONOBLOCK | TFD_CLOEXEC); 
if (fd == -1) {
    //handle_error ... 
}
```

`TFD_MONOBLOCK`, `TFD_CLOEXEC`这些参数和`MONOBLOCK`,`CLOEXEC`是一样的意思，使用`异或`来互相操作，默认问0。

### timerfd的使用例子

时钟的文件描述符使用`timerfd_create`来进行创建，创建成功后，调用`timerfd_settime()`来设置事件超时，时钟描述符主要使用一个`struct timerspec`的变量来初始化。

```
// 时钟超时间隔
struct timespec it_interval {
    time_t tv_sec 
    time_t tv_nsec
}

// 时钟超时时间
struct timespec it_value {
    time_t tv_sec
    time_t tv_nsec 
}
```

应用调用`read()`函数来等待时钟超时事件的到来即可，`read()`返回一个uint_64的整数，这个整数表示超时的次数，这些次数不会叠加，如：

```
/* This program prints "Beep!" every second with timerfd. */
#include <inttypes.h>
#include <stdio.h>
#include <sys/timerfd.h>
#include <unistd.h>

int main(void)
{
    int fd;
    struct itimerspec timer = {
        .it_interval = {1, 0},  /* 1 second */
        .it_value    = {1, 0},
    };
    uint64_t count;

    /* No error checking! */
    fd = timerfd_create(CLOCK_MONOTONIC, 0);
    timerfd_settime(fd, 0, &timer, NULL);
    int i = 0; 
    for (;;) {
        sleep(i);
        i++;
        read(fd, &count, sizeof(count));
        printf("Beep!\n");
        printf("count: %d, i:%d\n", count, i);
    }

    return 0;
}
```

输出：

```
Beep!
count: 1, i: 1
Beep!
count: 1, i: 2
Beep!
count: 2, i: 3
Beep!
count: 3, i: 4
Beep!
count: 4, i: 5
Beep!
count: 5, i: 6
```


## signalfd

也可以使用时钟信号的方式来实现定时事件......, 这个此处省略....
当然，其他信号也可以使用`epoll`来监听

## inotify 

这也不说了....

# timerfd, signalfd使用epoll处理

`epoll`的触发模式有：水平触发（电平触发） ， 边缘触发。

``` c
#include <inttypes.h>
#include <signal.h>
#include <stdio.h>
#include <sys/epoll.h>
#include <sys/timerfd.h>
#include <sys/signalfd.h>
#include <unistd.h>

int main(void)
{
    int efd, sfd, tfd;
    struct itimerspec timer = {
        .it_interval = {1, 0},  /* 1 second */
        .it_value    = {1, 0},
    };
    uint64_t count;
    sigset_t sigmask;
    struct signalfd_siginfo siginfo;
#define MAX_EVENTS 2
    struct epoll_event ev, events[MAX_EVENTS];
    ssize_t nr_events;
    size_t i;

    /* No error checking! */
    tfd = timerfd_create(CLOCK_MONOTONIC, 0);
    timerfd_settime(tfd, 0, &timer, NULL);

    sigfillset(&sigmask);
    sigprocmask(SIG_BLOCK, &sigmask, NULL);
    sfd = signalfd(-1, &sigmask, 0);

    // new a epoll instance
    efd = epoll_create1(0);

        // event type ==> read
    ev.events = EPOLLIN;
    // add timer fd
    ev.data.fd = tfd;
    epoll_ctl(efd, EPOLL_CTL_ADD, tfd, &ev);

    // event type => read
    ev.events = EPOLLIN;
    // add signal fd
    ev.data.fd = sfd;
    epoll_ctl(efd, EPOLL_CTL_ADD, sfd, &ev);

    for (;;) {
        nr_events = epoll_wait(efd, events, MAX_EVENTS, -1);
        for (i = 0; i < nr_events; i++) {
            if (events[i].data.fd == tfd) {
                read(tfd, &count, sizeof(count));
                printf("Beep!\n");
            } else if (events[i].data.fd == sfd) {
                read(sfd, &siginfo, sizeof(siginfo));
                printf("Received signal number %d\n", siginfo.ssi_signo);
            }
        }
    }
    return 0;
}
```

输出：

```
$ ./ep
Beep!
Beep!
Beep!
Beep!
Beep!
Beep!
^CReceived signal number 2
Beep!
Beep!
^CReceived signal number 2
Beep!
Beep!
Beep!
```

# 构建一个基于epoll的简单定时任务

首先需要构建一个简单的任务池，为了更好的并发，任务池需要一个任务队列，这个任务队列保存了`任务类型和任务数据（其实就是TaskFunc and FuncData）`.

## 构建 ThreadPool 


```
            |----------|
            |ThreadPool|
            |----------|
                  ↑
                  |
      ------------ ------------|           
      |                        |
      |                        |      
   |------|   ---------     |-------|             
   |Woker |   |  Init |     | free  |  
   | Job  |   |   Job |     | Job   |
   | Queue|   | Qeuue |     | Queue |  
   |------|   |-------|     |-------|  
       ↑                        ↑
       |                        |
       |----->|Job Thread|----->|

```


这个简单的线程池整体的设计就是这样，线程池对象包括主要包括三部分，`worker Job Queue`、`Free Job Queue`和`Job Thread`.

- Worker Job Queue

这是一个任务队列，这些任务队列有任务函数`job_cb`和数据参数`data`组成

- Free Job Queue 

这是一个空闲队列，这个队列主要是用来做任务回收（可以说是一个内存管理器）， 用户首先从空闲队列中获取一个`job_cb`和`data`指正,然后把自身的函数和数据赋给`job_cb`和`data`，在把组装好的任务放到`worker Job Queue`中。

```
TODO:

把 Init Job Queue去掉，并且把Free Job Queue交给一个内存管理线程进行管理
```

- Job Thread

`job thread` 是线程对象，这些线程对象首先会到`worker job queue`中获取到一个任务，执行完这个任务时，会把这个人回收到`free job queue`中



**source code**:

`header:`

``` c
/**/
#ifndef THREADPOOL_H_
#define THREADPOOL_H_
struct threadpool;
/**
This function creates a newly allocated thread pool
**/

struct threadpool * threadpool_init(int numbers);

int threadpool_add_job(struct threadpool *pool, void (*func )(void *), void *data, int blocking);

void threadpool_free(struct threadpool *pool, int blocking);
#endif /*THREADPOOL_H_*/

```

`source:`

``` c
#include "thr_pool.h"

#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>

#define LOG_DEBUG(s) printf("line %d,:%s, %s- \r\n", __LINE__, __func__, s); 
#define THREAD_POOL_QUEUE_SIZE 10

// queue iterm 
struct threadpool_job {
    void (*job_cb)(void*);
    void *data;
};  

struct threadpool_queue {
    unsigned int head;
    unsigned int tail;
    
    unsigned int iterm_size;
    
    void *iterms[THREAD_POOL_QUEUE_SIZE];
};

struct threadpool {
    struct threadpool_queue job_queue; // job queue 
    struct threadpool_queue free_job_queue; // free job queue 
    
    struct threadpool_job jobs[THREAD_POOL_QUEUE_SIZE];
    
    pthread_t *thr_arr;
    
    unsigned short thread_size; // thread size 
    volatile unsigned short stop_flag; // stop or start flag 
    
    pthread_mutex_t free_job_mutex; // free job queue 
    pthread_mutex_t mutex;  // 
    pthread_cond_t free_job_cond;
    pthread_cond_t cond;
} ;

/**
* This funtion inits a thread queue 
*
*@param queue The queue structure
*/

static void threadpool_queue_init(struct threadpool_queue *queue){
    int i ; 
    for (i =0; i < THREAD_POOL_QUEUE_SIZE; i ++){
        queue->iterms[i] = NULL;
    }
    
    queue->head = 0;
    queue->tail = 0;
    queue->iterm_size = 0;
}

/**
 * This function adds data to the tail of the queue.
 *
 * @param queue The queue structure.
 * @param data The data to be added to the queue.
 * @return 0 on success (the data was added to the queue) else -1 is returned.
 */

static int threadpool_queue_enqueue(struct threadpool_queue *queue, void *data) {
    if (queue->iterm_size == THREAD_POOL_QUEUE_SIZE) {
        LOG_DEBUG("The queue is full, unable to add data to it.");
        return -1;
    }
    
    if(queue->iterms[queue->tail] != NULL) {
        LOG_DEBUG("A problem was detected in the queque (expected NULL, but found a different value.");
        return -1;
    }
    
    queue->iterms[queue->tail] = data;
    queue->tail++;
    queue->iterm_size ++;
    LOG_DEBUG("Add Job\n");
    if(queue->tail == THREAD_POOL_QUEUE_SIZE) {
        queue->tail = 0;
    }
    return 0;
}

static void *threadpool_queue_dequeue(struct threadpool_queue *queque){
    void *data;
    
    if(queque->iterm_size == 0) {
        LOG_DEBUG("Tried to dequeue from an empty queque");
        return NULL;
    }
    
    data = queque->iterms[queque->head];
    queque->iterms[queque->head] = NULL;
    queque->iterm_size --;
    
    if (queque->iterm_size == 0){
        queque->head = 0;
        queque->tail = 0;
    }else{
        queque->head++;
        if(queque->head == THREAD_POOL_QUEUE_SIZE) {
            queque->head=0;
        }
    }
    
    return data;
}

static int threadpool_queue_is_empty(struct threadpool_queue *q){
    //return q->iterm_size?0:1;
    if (q->iterm_size==0){
        return 1;
    }
    return 0;
}

static int threadpool_queue_size(struct threadpool_queue *queque) {
    return queque->iterm_size;
}

static void threadpool_job_init(struct threadpool_job *job){
    job->data = NULL;
    job->job_cb = NULL;
}

/*
 * Ths function obtains a queued job from the pool and return it. 
 * if no such job is available the operation blocks. 
 *
 * @param pool the thread pool structure. 
 * @return a job or null error (or if thread pool should shut down ) . *
 * */
static struct threadpool_job * threadpool_get_job(struct threadpool *pool){
    struct threadpool_job *job;
    
    // lock threadpool
    if (pthread_mutex_lock(&(pool->mutex))){
        printf("pthread_mutex_lock error.\r\n");
        return NULL;
    }

    while(threadpool_queue_is_empty(&(pool->job_queue))){
            if (pool->stop_flag) {
                if(pthread_cond_broadcast(&(pool->cond))){
                    LOG_DEBUG("thread_cond_broadcast error.\r\n");
                }

                if (pthread_mutex_unlock(&(pool->mutex))){
                    LOG_DEBUG("pthread_mutex_unlock.\r\n");
                    return NULL;
                }
                return NULL;
            }
            // Block until a new job arrives .  
            if (pthread_cond_wait(&(pool->cond), &(pool->mutex))) {
                LOG_DEBUG("pthread_cond_wait.\r\n"); 
                if (pthread_mutex_unlock(&(pool->mutex))) {
                    LOG_DEBUG("pthread_mutex_unlock. \r\n");
                } 
                return NULL;
            }
            LOG_DEBUG("Wait for a job comming");
        }
        
        job = (struct threadpool_job*)threadpool_queue_dequeue(&(pool->job_queue));
    if (job== NULL){
        LOG_DEBUG("Failded to obtain a job from the job queue.");
    }
    
    if (pthread_mutex_unlock(&(pool->mutex))){
        LOG_DEBUG("pthread_mutex_unlock.");
        return NULL;
    }
    
    return job; 
}

static void *job_cb_func(void *data){
    struct threadpool *pool = (struct threadpool*)data;
    struct threadpool_job *job;
    LOG_DEBUG("do in thread ...");
    while(1){
        // take a job 
        LOG_DEBUG("REUEST A JOB");
        job = threadpool_get_job(pool);
        LOG_DEBUG("GOT A JOB");
        if (job == NULL) {
            if (pool->stop_flag) {
                // job thr need to exit (thread pool was shutdown)
                break;
            }else {
                // An error has occurented.
                LOG_DEBUG("a error happend when trying to obtain a work job.");
                LOG_DEBUG("The job thread has exited.");
                break;
            }
        }
        
        if(job->job_cb){
            job->job_cb(job->data);
            
            // release the job by returing it to the free job queue.
            threadpool_job_init(job);
            
            if (pthread_mutex_lock(&(pool->free_job_mutex))) {
                LOG_DEBUG("pthread mutex lock error.");
                LOG_DEBUG("The job thread has exited.");
                break;
            }
            
            // put the idle job into thread pool's free job queue   
            if (threadpool_queue_enqueue(&(pool->free_job_queue), job)) {
                LOG_DEBUG("Failed to enqueue a job to free job queue.");
                if (pthread_mutex_unlock(&(pool->free_job_mutex))) {
                    LOG_DEBUG("pthread mutext unlock error.");
                }
                
                LOG_DEBUG("The Job thread has exited."); 
                break;
            }
            
            if (threadpool_queue_size(&(pool->free_job_queue)) ==1) {
                // Notify all waiting threads that new tasks can added 
                if (pthread_cond_broadcast(&(pool->free_job_cond))) {
                    LOG_DEBUG("pthread cond broadcast . "); 
                    
                    if (pthread_mutex_unlock(&(pool->free_job_mutex))) {
                        LOG_DEBUG("pthread mutex unlock.");
                    }
                    
                    break;
                }
            }
            
            if (pthread_mutex_unlock(&(pool->free_job_mutex))) {
                LOG_DEBUG("The job thread has exited."); 
                break;
            }
        }
    }   

    LOG_DEBUG("Thread Exit ...");
    
    return NULL;
}

static void *stop_job_thr_func_cb(void *ptr) {
    int i ; 
    struct threadpool *pool = (struct threadpool *)ptr;
    
    if (pthread_mutex_lock(&(pool->mutex))) {
        LOG_DEBUG("Warning: Memory was not released .");
        LOG_DEBUG("Warning: Some of the job threads may have failed to exit.");
        return NULL;
    }
    
    pool->stop_flag = 1; 
    
    while(!threadpool_queue_is_empty(&(pool->job_queue))){
        LOG_DEBUG("Wait All Sub thread exit...");
        // Blocking until all job have been executed, safely stop the thread pool .
        if (pthread_cond_wait(&(pool->cond), &(pool->mutex))){
            LOG_DEBUG("pthread cond wait.");
            if (pthread_mutex_unlock(&(pool->mutex))){
                LOG_DEBUG("pthread mutex unlock.");
            }
            return NULL;
        }
    }
    
    LOG_DEBUG("Wait All Sub thread exit...");
    // Wakeup all job threads (broadcast operation). 
    if (pthread_cond_broadcast(&(pool->cond))){
        LOG_DEBUG("Warning: Memory was not released."); 
        LOG_DEBUG("Warning: Some of the job threads my have failed to exit .");
        return NULL;
    }

    if (pthread_mutex_unlock(&(pool->mutex))){
        LOG_DEBUG("pthread mutex unlock error.");
        return NULL;
    }

    // Wait until all job threads are done. 
    for (i = 0; i < pool->thread_size; i ++) {
        LOG_DEBUG("Wait sub thread exit..");
        if (pthread_join(pool->thr_arr[i], NULL)) {
            LOG_DEBUG("pthread join error.");
        }
    }
    
    // Free all allocated memory 
    free(pool->thr_arr);
    free(pool);
    
    return NULL;    
}

struct threadpool * threadpool_init( int threads_size){
    struct threadpool *pool; 
    int i; 
    
    // Create the thread pool struct. 
    pool = malloc(sizeof(struct threadpool));
    if (pool == NULL) {
        LOG_DEBUG("malloc thread pool error.");
        return NULL;
    }
    
    pool->stop_flag = 0; 
    
    // Init the mutex and cond vars 
    if (pthread_mutex_init(&(pool->free_job_mutex), NULL)) {
        LOG_DEBUG("pthread mutex init ."); 
        free(pool);
        return NULL;
    }
    if (pthread_mutex_init(&(pool->mutex), NULL)) {
        LOG_DEBUG("pthread mutex init error"); 
        free(pool);
        return NULL;
    }
    if (pthread_cond_init(&(pool->free_job_cond), NULL)) {
        LOG_DEBUG("pthread cond init error");
        free(pool);
        return NULL;
    }
    if (pthread_cond_init(&(pool->cond), NULL)) {
        LOG_DEBUG("pthread cond init error"); 
        free(pool);
        return NULL;
    }
    
    // Init the jobs queue 
    threadpool_queue_init(&(pool->job_queue)); 
    
    // Init the free job queue . 
    threadpool_queue_init(&(pool->free_job_queue)); 
    
    // Add all the free job to the free job queue 
    for (i = 0; i < THREAD_POOL_QUEUE_SIZE; i++){
        threadpool_job_init((pool->jobs) + i);
        if (threadpool_queue_enqueue(&(pool->free_job_queue), (pool->jobs) +i)) {
            LOG_DEBUG("Failed to a job to the free job queue during initialization."); 
            free(pool);
            return NULL;
        }
    }

    printf("The free Queue is : %d\n", pool->free_job_queue.iterm_size);
    printf("The Worker Queue size is: %d\n", pool->job_queue.iterm_size);

    
    // Create the thr_arr. 
    pool->thr_arr = malloc(sizeof(pthread_t) * threads_size);
    if (pool->thr_arr == NULL) {
        LOG_DEBUG("malloc thread arr error."); 
        free(pool);
        return NULL;
    }
    
    // Start the job threads 
    for (pool->thread_size = 0; pool->thread_size < threads_size; (pool->thread_size ++)) {
        if (pthread_create(&(pool->thr_arr[pool->thread_size]), NULL, job_cb_func, pool)) {
            LOG_DEBUG("pthread create function error."); 
            threadpool_free(pool, 0); 
            return NULL;
        }
    }
    
    return pool;
}

int threadpool_add_job (struct threadpool *pool, void (*job_cb)(void *), void *data, int blocking){
    struct threadpool_job * job ; 
    if (pool == NULL) {
        LOG_DEBUG("The thread pool is null"); 
        return -1;
    }
    
    // lock free job queque
    if (pthread_mutex_lock(&(pool->free_job_mutex))){
        LOG_DEBUG("pthread mutext lock err."); 
        return -1;
    }
    
    // Check if the free job queque is empty 
    while(threadpool_queue_is_empty(&(pool->free_job_queue))) {
        // The job is empty 
        LOG_DEBUG("The Free Queue Size equal zero");
        if(!blocking) {
            // Return immediately if the command is non blocking 
            if (pthread_mutex_unlock(&(pool->free_job_mutex))) {
                LOG_DEBUG("pthread mutex unlock."); 
                return -1;
            }
            return -2;
        }
        
        // blocking is set to 1, wait until free job queue has a job to obtain 
        if (pthread_cond_wait(&(pool->free_job_cond), &(pool->free_job_mutex))) {
            LOG_DEBUG("pthread cond mutex."); 
            if (pthread_mutex_unlock(&(pool->free_job_mutex))) {
                LOG_DEBUG("pthread mutex unlock."); 
            }
            return -1;
        }
    }
    
    // Obtain an empty job. 
    job = (struct threadpool_job*) threadpool_queue_dequeue(&(pool->free_job_queue));
    if (job == NULL) {
        LOG_DEBUG("Failed to obtain an empty job from the free job queue."); 
        if (pthread_mutex_unlock(&(pool->free_job_mutex))) {
            LOG_DEBUG("pthread mutex unlock.");
        }
        
        return -1;
    }
    
    if (pthread_mutex_unlock(&(pool->free_job_mutex))){
        LOG_DEBUG("pthread mutex unlock."); 
        return -1;
    }
    
    job->data = data; 
    job->job_cb = job_cb; 
    
    // Add the job into the job queue 
    if (pthread_mutex_lock(&(pool->mutex))) {
        LOG_DEBUG("pthread mutex lock."); 
        return -1;
    }
    
    if (threadpool_queue_enqueue(&(pool->job_queue), job)){
        LOG_DEBUG("Failed to add new job to the job queue."); 
        if (pthread_mutex_unlock(&(pool->mutex))) {
            LOG_DEBUG("pthread mutex unlock."); 
        }
        return -1;
    }
    
    if (threadpool_queue_size(&(pool->job_queue)) == 1) {
        // Notify all worker threads that there are new jobs 
        if (pthread_cond_broadcast(&(pool->cond))) {
            LOG_DEBUG("pthread cond broadcast error"); 
            if (pthread_mutex_unlock(&(pool->mutex))) {
                LOG_DEBUG("pthread mutex unlock error");
            }
            return -1;
        }
    }
    if (pthread_mutex_unlock(&(pool->mutex))) {
        LOG_DEBUG("pthread mutex unlock error");
        return -1; 
    } 
    return 0;
}

void threadpool_free(struct threadpool *pool, int blocking){
    pthread_t thr; 
    if (blocking) {
        stop_job_thr_func_cb(pool);
    }else{
        if (pthread_create(&thr, NULL, stop_job_thr_func_cb, pool)) {
            LOG_DEBUG("pthread create function error.");
            pool->stop_flag = 1;
        }       
    }
}
```

`test Code`:

``` c
#include "thr_pool.h"
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>

#define ARR_SIZE 1000000

static pthread_mutex_t count_mutex = PTHREAD_MUTEX_INITIALIZER; 
static int count; 

void fast_job1(void *ptr) {
    int *pval = (int*)ptr;
    int i; 
    printf("Do in fast_job1");
    for (i =0; i < 1000; i ++) {
        (*pval) ++;
    }
    
    printf("ptr:%d\r\n", pval);    
    pthread_mutex_lock(&count_mutex);
    count++; 
    pthread_mutex_unlock(&count_mutex);
}

void slow_job(void *ptr) {
    printf("slow job: count value is %d\n", count); 
    
    pthread_mutex_lock(&count_mutex);
    
    count ++; 
    
    pthread_mutex_unlock(&count_mutex);
}


int main (){
    struct threadpool *p; 
    
    int arr[ARR_SIZE], i, ret, failed_count =0; 
    for (i =0; i < ARR_SIZE; i++){
        arr[i] = i;
    }
    
    p = threadpool_init(200) ; 
    if (p==NULL) {
        printf("Error! Failed to create a threadpool struct \n"); 
        exit(EXIT_FAILURE);
    }
        
    
    for (i =0 ; i < ARR_SIZE; i ++) {
        if (i % 2==0){
            ret = threadpool_add_job(p, slow_job, arr +i , 1);
        }else {
            ret = threadpool_add_job(p, fast_job1, arr+i,  0);
        }
        
        if (ret == -1) {
            printf("An error had occurred while adding a job, %d\r\n", ret);
            exit(EXIT_FAILURE);
        }
        if (ret == -2 ){
            failed_count ++;
        }
    }
    threadpool_free(p, 1);
    // stop the pool 
    printf("Example ended.\n");
    printf("%d tasks out of %d have been executed.\n",count,ARR_SIZE);
    printf("%d tasks out of %d did not execute since the pool was overloaded.\n",failed_count,ARR_SIZE);
    printf("All other tasks had not executed yet.");
    return 0;

}
```

## 构建一个eventloop对象

好吧，暂时没时间，先这样.......

以后有时间再能吧.


资料来源：
- http://www.enodev.fr/posts/notifications-with-file-descriptors.html
- http://docs.oracle.com/cd/E19253-01/816-5137/ggedn/index.html