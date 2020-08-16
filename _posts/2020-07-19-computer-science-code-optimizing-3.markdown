---
layout: post
title:  "[计算机技术] 代码优化3-并行代码优化"
date:   2020-07-19 02:24:00 +0800
comments: true
categories: [计算机技术, 优化, 并行, OPENMP, CUDA]
mathjax: true
---

* TOC
{:toc}

> "Premature optimization is the root of all evil."

### 概述

首先本章讨论的并行是指线程级别的并行，关于指令级别并行(**ILP**) 和数据并行可以参考[线性代码优化]({% post_url 2020-04-07-computer-science-code-optimizing-1 %})。而线程级别的并行主要分为在同一个cpu核上跑多个线程的并发编程和跑在多个cpu核上的并行编程，虽然说从写代码的角度一般不太需要特别关心具体是哪一种类型，不过这两者之间有一个比较重要的区别：同一个核上的线程共享同一个**L1 cache**，而不同核之间的线程只共享**L2 cache**。另外，一般每个核有2个计算单元，所以一般代码中创建的逻辑上的线程个数是机器物理cpu个数的两倍。对每个线程而言，他们分别有各自的_寄存器_，*PC(program counter)*，*栈内存（区别于堆内存）*。而不同的线程共享*同一份代码*，*静态内存*。

```zsh
$ sysctl -a | grep hw.logicalcpu
hw.logicalcpu: 8
```

另外，为什么需要并行优化而不是继续提高单个程序的运行速度？这是因为如下图所示，其实cpu核的频率增长速度已经近乎停滞了，所以线性代码很难在硬件上继续实现加速了，因此，多核，并行的硬件加速应运而生。

<div style="text-align:center;"><img src="/assets/opt-3/trend.png" width="60%" height="60%"></div>

 接下来主要介绍两大部分：cpu并行和gpu并行。cpu并行主要介绍了并行中的数据存取，OpenMP，缓存一致性，线程安全的queue，线程池和多个进程之间的通信。gpu主要介绍cuda线程的硬件分配方式，另外介绍了2个常用的用例，矩阵运算和`reduce`操作，最后介绍了常用的库`thrust`。

### 基于CPU的代码优化

#### Amdahl's law

假设一份纯线性的代码，其中占运行速度$$F=25\%$$的部分被并行化，并行化之后速度变为原来的$$S=2$$倍，那么请问最后代码的速度变成原来的几倍？

带入下面公式

$$ Speedup = \frac{1}{1-F+\frac{F}{S}} $$

得到$$Speedup=1.14$$，也就是和原代码相比，仅仅提高了0.14倍的速度，上面的公式就是Amdahl's law，它衡量了一份代码并行优化之后速度的提升。

这个法则的指导意义在于告诉我们，代码的速度瓶颈主要取决于线性部分的代码占总代码运行时间的多少，而并行代码的优化的倍数$$S$$往往并不会对运行速度造成太大的影响。因此，优化代码时主要需要考虑的是如何减少线性部分的代码，而非把并行的性能提升到极致。

#### 数据竞速，锁和原子性

**数据竞速**：由于不同线程对同一个地址的写入顺序是不确定的，所以导致这个地址最终的结果每次运行会出现不一样的结果。

避免的方法是1.不要对同一个地址同时进行写入，2.保证读取数据和写入数据同步，从而保证结果确定。

**锁**：锁就是用来同步保证避免发生数据竞速的方案。

```c++
while (lock == 1) {}
lock = 1;
// critical section
lock = 0;
```

这里需要指出，锁的实现**只能通过原子操作**，所以**永远不要自己设计锁**，具体的原因比较啰嗦就不具体介绍了，感兴趣的google上其实很多。

任何写入的指令都需要先读取，再写入，需要两步，这个过程中就会导致自己设计的锁存在bug，**原子操作指令**最大的特点在于，读取和写入只需要一条指令（更准确的说是compare and exchange），从而可以实现锁的设计。

最后，我认为锁就像易筋经，忘的越多，功力越强；懂得越多，反而越容易出错，原子操作其实还涉及到memory ordering的内容，不过据[C++ Concurrency in Action]([https://github.com/huyubing/books-pdf/blob/master/C%2B%2B%20Concurrency%20in%20Action.pdf](https://github.com/huyubing/books-pdf/blob/master/C%2B%2B Concurrency in Action.pdf))的说法，世界上懂memory ordering的人不超过五国，所以最好的办法是不用管他。很多人自以为理解了memory ordering，但是实际上没有，就会导致为了一点微乎其微的效率去修改原子操作的memory ordering但写的代码逻辑有问题导致出现及其隐蔽的bug。

#### OpenMP

OpenMP是一个为多线程服务的共享内存模式的并行api。它是基于C的，所以使用起来非常方便，不需要学复杂的用法。使用时只需要保证

1.C或者C++头文件包含`#include <omp.h>`.

2.编译时需要加flag：`gcc -fopenmp`(在mac系统的nnvm编译器里需要加`-Xpreprocessor -fopenmp  -lomp`)

OpenMP的工作原理是开始执行程序时，只有一个主线程线性执行，直到遇到第一个并行块区域，然后他会执行**fork**，也就是创建很多个并行线程。所有线程运行完后，主线程会进行**join**操作，也就是同步这些线程并且结束他们。

OpenMP的基本结构如下：

```c++
#include <omp.h>
#pragma omp parallel //[private(i)] [shared(share_val)]
{  // 这个大括号必须出现在这个位置，不可以接着上一行的命令
	/* parallel codes */
}
```

其中，private语句声明每个线程私有的变量名，shared声明每个线程共享的变量名。默认在线程外定义的变量都是共享的。

`OMP_NUM_THREADS`环境变量决定了会创建多少个线程，在每个线程中，可以通过如下指令读取或设置线程数。

```c++
omp_set_num_threads(x);  // 设置线程个数
num_th = omp_get_num_threads();  // 获得线程个数
th_ID = omp_get_thread_num();  // 获得当前线程id
```

那么，一个OpenMP的并行hello world代码就如下

```c++
#include <stdio.h>
#include <omp.h>
int main() {
  int nthreads, tid;
  /* fork线程，并且设置tid为每个线程的私有变量 */
#pragma omp parallel private(tid)
  {
    tid = omp_get_thread_num(); /* 获得当前线程id */
    printf("Hello World from thread = %d\n", tid);
    /* 仅有主线程会执行下面代码 */
    if (tid == 0) {
      nthreads = omp_get_num_threads();
      printf("Number of threads = %d\n", nthreads);
    }
  }  // 默认所有的线程都会同步，然后join到主线程，并且结束
}
```

接下来介绍一些常用的OpenMP的指令，首先是并行for循环

```c++
int i;
#pragma omp parallel for [schedule(type [, chunk])]
for (i = 0; i < len; ++i) {
	/* parallel codes */
	for (int j = 0; j < i; ++j) {  // 双重循环时内部循环不并行
		/* inner loop codes */
	}
}
```

OpenMP会把循环拆成和线程个数相等的**chunks**，例如`len = 100`并且有2个线程时，**chunk0**就是0-49，**chunk1**就是50-99。另外不允许循环里有能中途退出循环的语句，如`break`,`return`,`go to`等。

另外需要介绍一下可选的参数`schedule`。括号里`type`常用的为`static`或者`dynamic`，chunk为每个线程分配到的循环次数，默认为平均分配给每个线程对应的数值。`static`情况下，所有线程执行完`chunk`次循环后，再执行下一轮剩下的对应循环；而`dynamic`情况下，每个线程执行好了，就会直接开始执行下一个`chunk`的循环，不会等待别的线程。

当循环中需要做`reduction`操作时，可以在指令后面加`reduction (operator: var)`，例如

```c++
int i, result = 0;
#pragma omp parallel for reduction(+: result)
for (i = 0; i < n; ++i)
  result = result + (a[i] * b[i]);
```

另外常见的指令还有：

`sections`用于规定每个线程需要执行哪些任务

`single`指定并行程序中只会被执行一次的代码

`critical`保证代码中的指令同一时间只有一份线程在运行，和锁的作用一样。

#### 并行中的缓存一致性和false sharing

当线程被分配到不同的处理器核上时，每个线程都有自己的缓存，此时就会出现一个问题。例如对于共享变量`a`，线程B先读取`a`到缓存中，然后线程A对`a`进行写入，而线程B如果直接读取 `a`，就会得到缓存中的未被修改的值，这时就会发生错误。所以多线程中需要保证缓存是一致的。

解决方案是，每当一个线程对一个内存地址写入时，通过总线通知其他所有缓存，这个地址的值变得无效了，也就是把对应缓存的`valide`位设为0（这个位的另一个作用是初始化时告诉代码这一段内存和数值是随机初始化的无用值还是已经被初始化的正确值），这样别的线程再去读取时就会发生**cache miss**，然后去内存重新读取，这样的操作称作**snooping**。

回到之前的例子，当线程A不断的对地址为4000位置的内存进行读写，而线程B不断对地址位置为4012的内存进行读写，而cache block的size是36byte时，**snooping**会导致什么？它们会不断的发生**cache miss**并不断的读写内存值，仿佛cache不存在了，这个问题就叫做**false sharing**，也叫做**block ping-pong**。所以一般并行for语句时，一个线程总是执行相邻的index，而不是以**thread number**为stride跳跃着执行循环。

#### C++11中的线程池实现

上面介绍了一些比较基础的并行，有时候我们每个任务的时间差异很大，需要并行执行的任务也很多，因此下面介绍线程安全的Queue和线程池。线程池接受不断到来的需要并行执行的函数，然后把它们push进Queue中，而工作线程则会不断查询Queue是否有函数需要执行，有就执行完。

对用户而言，用户只需把需要并行执行的函数发送给线程池就可以。

##### 线程安全的Queue

线程安全的Queue主要通过锁和条件变量构成，push时上锁，并发送条件变量；pop有2种选项，第一种`try_pop`，成功则返回`true`，失败返回`false`（Queue为空），第二种`wait_and_pop`，当Queue为，代码如下

```c++
#ifndef THREADSAFE_QUEUE_H
#define THREADSAFE_QUEUE_H
#include<thread>
#include<queue>
#include<exception>
#include<iostream>
#include<mutex>
#include<condition_variable>

template<typename T>
class threadsafe_queue
{
private:
    mutable std::mutex mut;
    std::queue<T> data_queue;
    std::condition_variable data_cond;
public:
    threadsafe_queue() {}
    void push(T new_value)
    {
        std::lock_guard<std::mutex> lk(mut);
        data_queue.push(std::move(new_value));
        data_cond.notify_one();
    }
    
    void wait_and_pop(T &value)
    {
        std::unique_lock<std::mutex> lk(mut);
        data_cond.wait(lk, [this]{return !data_queue.empty();});
        value = std::move(data_queue.front());
        data_queue.pop();
    }

    std::shared_ptr<T> wait_and_pop() 
    {
        std::unique_lock<std::mutex> lk(mut);
        data_cond.wait(lk, [this]{return !data_queue.empty();});
        std::shared_ptr<T> res(std::make_shared<T>(std::move(data_queue.front())));
        data_queue.pop();
        return res;
    }
    std::shared_ptr<T> try_pop() 
    {
        std::unique_lock<std::mutex> lk(mut);
        if (data_queue.empty())
            return std::shared_ptr<T>();
        std::shared_ptr<T> res(std::make_shared<T>(std::move(data_queue.front())));
        data_queue.pop();
        return res;
    }
    bool try_pop(T &value)
    {
        std::unique_lock<std::mutex> lk(mut);
        if (data_queue.empty())
            return false; 
        value = std::move(data_queue.front());
        data_queue.pop();
        return true;
    }
    bool empty() const
    {
        std::lock_guard<std::mutex> lk(mut);
        return data_queue.empty();
    }
};

#endif // THREADSAFE_QUEUE_H
```

#####  线程池

线程池创建工作线程，然后维护一个元素为函数的成员变量`worker_queue`。代码如下

```c++
#ifnef THREAD_POOL_H
#define THREAD_POOL_H
#include "threadsafe_queue.h"

#include <iostream>
#include <functional>
#include <atomic>
#include <thread>


class join_threads
{
    std::vector<std::thread> &threads;
public:
    explicit join_threads(std::vector<std::thread> &threads_):
        threads(threads_)
    {}
    ~join_threads()
    {
        for(unsigned long i = 0; i < threads.size(); ++i)
        {
            if (threads[i].joinable())
                threads[i].join();
        }
    }
};

class thread_pool
{
    std::atomic_bool done;
    threadsafe_queue<std::function<void()> > worker_queue;
    std::vector<std::thread> threads;
    join_threads joiner;
    void worker_thread()
    {
        while(!done)
        {
            std::function<void()> task;
            if(worker_queue.try_pop(task))
            {
                task();
            }
            else
            {
                std::this_thread::yield();
            }
        }
    }
public:
    thread_pool():
        done(false), joiner(threads)
    {
        unsigned const thread_count = std::thread::hardware_concurrency();
        try
        {
            for (unsigned i = 0; i < thread_count; ++i)
            {
                threads.push_back(
                        std::thread(&thread_pool::worker_thread, this));
            }
        }
        catch(...)
        {
            done = true;
            throw "an error occured!\n";
        }
    }
    ~thread_pool()
    {
        done = true;
    }
    template<typename FunctionType>
    void submit(FunctionType f)
    {
        worker_queue.push(std::function<void()>(f));
    }
};

#endif // THREAD_POOL_H
```

用户使用时，只需要需要执行下面指令

 ```c++
#include "thread_pool.h"

#include <iostream>
#include <functional>
#include <atomic>
#include <thread>  // 必须包含这个头文件

std::atomic<int> cnt;  // 原子变量
void task() {
    cnt.fetch_add(10);
    std::cout<<cnt.load()<<std::endl;
}
int main() {
    thread_pool tp;
    for (int i = 0; i < 100; ++i)
    {
        std::function<void()> f = task;
        tp.submit(f);  //提交需要执行的函数
    }
}
 ```

#### OpenMPI和进程间通信



---

### 基于cuda的代码优化

#### cuda硬件线程分配原理

#### 矩阵操作实例

#### `reduce_op`操作实例

#### thrust介绍

