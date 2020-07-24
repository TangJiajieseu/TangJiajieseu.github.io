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



#### 并行中的缓存一致性

#### 线程安全的Queue

#### 线程池

#### OpenMPI和进程间通信

---

### 基于cuda的代码优化

#### cuda硬件线程分配原理

#### 矩阵操作实例

#### `reduce_op`操作实例

#### thrust介绍

