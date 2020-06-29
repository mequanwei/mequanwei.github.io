---
title: 线程同步和线程池
abbrlink: 2825544451
date: 2020-04-05 08:40:33
tags:
---
在上次的基础上，将进程模型改为线程模型，并添加了线程池。项目地址 [github](https://github.com/mequanwei/wServer)
<!--more-->

上次的服务器模型采用子进程监听新的socket，考虑到在进程线程数量少的时候，二者并没有太大差异，所以本次将进程模型替换成线程模型，并且实现了线程池机制。

## 线程间同步
在多线程环境中运行的代码段，需要考虑线程间同步。对于一些临界区资源，要保证临界区的同一时刻仅被一个线程操作，就需要用到线程间的互斥-锁机制，线程间同步的机制有信号量，互斥量，条件变量。C++11之后提供了thread线程类，可以很方便的使用这些机制。不过由于我的系统 gcc 版本过低，项目中并没有用它。而是使用了 linux 原生线程支持。等以后有空在慢慢优化。

### 互斥锁mutex
mutex 是最基本的互斥量，提供了独占所有权的特性，对其操作包括：
```c++
pthread_mutex_init(): 初始化互斥锁

pthread_mutex_destory(): 销毁互斥锁

pthread_mutex_lock():加锁

pthread_mutex_unlock():互斥锁解锁
```

### 条件变量

条件变量提供了一种线程间的通知机制,当某个共享数据达到某个值时,唤醒等待这个共享数据的线程。资料上显示条件变量的效率比 mutex 高，有机会可以测试一下。
```c++
pthread_cond_init(): 初始化条件变量

pthread_cond_destory(): 销毁条件变量

pthread_cond_broadcast(): 函数以广播的方式唤醒所有等待目标条件变量的线程

pthread_cond_wait(): 函数用于等待目标条件变量.该函数调用时需要传入 mutex 参数。执行时,先把调用线程放入条件变量的请求队列,然后将互斥锁mutex解锁,当函数成功返回为0时,互斥锁会再次被锁上。也就是说函数内部会有一次解锁和加锁操作.
```

### 信号量
信号量是一种特殊的变量，它只能取自然数值并且只支持两种操作：等待(P)和信号(V)。

```c++
sem_init():初始化一个未命名的信号量

sem_destory():用于销毁信号量

sem_wait():将信号量减一,信号量为0时,sem_wait 阻塞

sem_post():将信号量加一,信号量大于0时,唤醒调用sem_post的线程
```

在项目中，我们对这些基本操作进行了封装，主要是将锁初始化和销毁封装到构造函数和析构函数，这样当对象析构时，锁就会自动销毁。以互斥量为例：
```c++
Mutex::Mutex()
{
	if (pthread_mutex_init(&m_mutex, NULL) != 0)
	{
		throw std::exception();
	}
}

Mutex::~Mutex()
{
	pthread_mutex_destroy(&m_mutex);
}

bool Mutex::Lock()
{
	return pthread_mutex_lock(&m_mutex) == 0;
}

bool Mutex::Unlock()
{
	return pthread_mutex_unlock(&m_mutex) == 0;
}

```

## 线程池
随着服务器的运行，系统接收到 socket，服务端关闭 socket 要结束线程。线程过多或者频繁创建和销毁线程会带来调度开销，进而影响缓存局部性和整体性能。为了避免这种情况，我们用到了线程池。线程池维护着多个线程，等待着管理器分配可并发执行的任务。

线程池的主要有两部分——工作线程和等待队列。其基本思想是，初始化一定数量的线程，这些线程不断去工作队列中取出任务并执行。当工作队列没有任务时，线程池主阻塞等待。当往工作队列中添加一个任务时，通过信号量的方式通知线程池开始工作。其主要代码如下：
```c++

template <typename T>
ThreadPool<T>::ThreadPool(ConnectionPool& connPool, int thread_number, int max_requests):m_thread_number(thread_number), m_max_requests(max_requests), m_threads(NULL),m_connPool(&connPool)
{
    if (thread_number <= 0 || max_requests <= 0)
        throw std::exception();
    m_threads = new pthread_t[m_thread_number];
    if (!m_threads)
        throw std::exception();
    for (int i = 0; i < thread_number; ++i)
    {
        if (pthread_create(m_threads + i, NULL, worker, this) != 0)
        {
            delete[] m_threads;
            throw std::exception();
        }
        if (pthread_detach(m_threads[i]))
        {
            delete[] m_threads;
            throw std::exception();
        }
    }
}


template <typename T>
bool ThreadPool<T>::append(T *request, int state,int fd)
{
    m_queuelocker.Lock();
    if (m_workqueue.size() >= m_max_requests)
    {
        m_queuelocker.Unlock();
        return false;
    }
    request->rw_state = state;
    request->fd = fd;
    m_workqueue.push_back(request);
    m_queuelocker.Unlock();
    m_queuestat.post();
    return true;
}

template <typename T>
void *ThreadPool<T>::worker(void *arg)
{
	ThreadPool *pool = (ThreadPool *)arg;
    pool->run();
    return pool;
}

template <typename T>
void ThreadPool<T>::run()
{
    while (true)
    {
        m_queuestat.wait();
        m_queuelocker.Lock();
        if (m_workqueue.empty())
        {
            m_queuelocker.Unlock();
            continue;
        }
        T *request = m_workqueue.front();
        m_workqueue.pop_front();
        m_queuelocker.Unlock();
        if (!request)
        {
        	continue;
        }

//		m_connPool = &request->mysql;
        if (request->rw_state == 0)
        {
        	request->readHandle(); //read
        }else
        {

        }

    }
}
```

在构造函数中，我们通过循环初始化若干工作线程。由于我是4核机器，这里的工作线程数量取 4。构造完成的线程池处于一个while循环中，通过信号量 m_queuestat 等待工作。当通过 append 向工作队列添加一个任务时，将 m_queuestat 信号量加一。这样线程池就开始从工作队列中取出一个元素，开始执行任务。
需要注意的是：操作等待队列时需要加互斥锁。这个简单的线程池并没有没有实现线程的优先级调度，这个后面有空也可以加上。