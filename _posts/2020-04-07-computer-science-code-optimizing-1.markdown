---
layout: post
title:  "[计算机技术] 代码优化1-线性代码优化"
date:   2020-04-07 01:43:52 +0800
comments: true
categories: [计算机技术, 优化, 循环展开, intel架构]
mathjax: true
---

* TOC
{:toc}

> "Premature optimization is the root of all evil."

## 概述

本章详细介绍线性代码优化中的一些技巧，第一部分介绍常用的编程时应该注意的可能降低效率的一些注意点；第二部分介绍基于cpu架构进行优化的技巧，主要内容是循环展开相关的技巧；第三部分简述现代cpu常用的**simd**编程技术。另外介绍每一个点的时候会讨论为什么现代编译器不会帮我们做这些工作。

由于所有的优化目标都是在不显著提高内存占用的前提下提高运行速度，代码优化的指标是**CPE**(cycles per element)，也就是每次循环迭代需要多少个计算机时钟周期，我的cpu型号是*Intel(R) Core(TM) i7-4770HQ CPU @ 2.20GHz*，对应的是*Haswell*微架构。每次个时钟周期为1/2.2纳秒。

这里优化的例子原始版本非常简单，代码如下

```c++
void combine1(vec_ptr v, data_t *dest) {
    long i;
    *dest = IDENT;
    for (i = 0; i < vec_length(v); ++i) {
        data_t val;
        get_vec_element(v, i, &val);
        *dest = *dest OP val;
    }
}
```

即对一个整数或者浮点数类型的*vector*，将其中的元素按照*OP*（加法或乘法）运算把结果储存于dest中。朴素版本和开了一级编译优化的**CPE**如下表

朴素版本的**CPE**如下表所示

|   函数   | 选项 | long add | long mul | double add | double mul |
| :------: | :--: | :------: | :------: | :--------: | :--------: |
| combine1 |  无  |   15.5   |   15.9   |    15.8    |    16.0    |
| combine1 | -O1  |   7.3    |   8.5    |    7.3     |    7.0     |

接下来的优化都会在`-O1`的基础上进行，并且分析为什么这样的优化不会被编译器执行。

### 常见编程优化

#### 消除多余计算

显然循环中的`vec_length(v)`这一函数的调用是多余的，将`length`的值在循环外一次求好，不需要反复调用，优化成如下

```c++
void combine2(vec_ptr v, data_t *dest) {
    long i;
    *dest = IDENT;
    long length = vec_length(v);
    for (i = 0; i < length; ++i) {
        data_t val;
        get_vec_element(v, i, &val);
        *dest = *dest OP val;
    }
}
```

改进的**CPE**如下表

|   函数   |       选项       | long add | long mul | double add | double mul |
| :------: | :--------------: | :------: | :------: | :--------: | :--------: |
| combine1 |       -O1        |   7.3    |   8.5    |    7.3     |    7.0     |
| combine2 | 移除`vec_length` |   4.8    |   6.0    |    6.5     |    8.1     |

一个更极端的情况是在处理字符串`s`时，会有如下循环代码

```
for (i = 0; i < strlen(s); ++i)
```

`strlen(s)`的时间复杂度是`O(n)`，如果不把这一步移除循环，那么每次迭代都会额外浪费`O(n)`的时间。

另一个需要讨论的问题是，为什么编译器不会帮我们优化？这是因为`vec_length`调用多少次会产生怎样的副作用是非常难以被编译器预测的，例如如果`vec_length`中有如下代码

```c++
static a = 0;
a += 1;
```

那么这个函数调用次数不同就会导致不同的结果，而编译器很难预测函数调用次数不同会导致怎样的后果，因此不会对此进行优化。

另外，为了减少函数调用，也可以将`get_vec_element`函数直接替换为对向量中数组的访问

```c++
void combine3(vec_ptr v, data_t *dest) {
    long i;
    *dest = IDENT;
    data_t *data = v->data;
    long length = vec_length(v);

    for (i = 0; i < length; ++i) {
        *dest = *dest OP data[i];
    }
}
```

改进后速度如下表

|   函数   |         选项          | long add | long mul | double add | double mul |
| :------: | :-------------------: | :------: | :------: | :--------: | :--------: |
| combine2 |   移除`vec_length`    |   4.8    |   6.0    |    6.5     |    8.1     |
| combine3 | 移除`get_vec_element` |   4.8    |   6.0    |    6.0     |    7.6     |

速度并没有什么影响，主要是因为函数访问本身只增加常数级别的寄存器压栈和退栈操作。

#### 移除不必要的内存访问

`combine3`中，每次循环里都会对`dest`进行一次读取和写入，而`dest`是一个内存地址，每次读取写入都会访问其对应的缓存，对内存的访问速度是远远慢于寄存器的访问的，因此可以用临时变量保存结果，最终将临时变量赋值给`dest`。另外`get_vec_element`也可以省去，对程序速度几乎没有影响，最终代码如下

```c++
void combine4(vec_ptr v, data_t *dest) {
    long i;
    *dest = IDENT;
    data_t *data = v->data;
    long length = vec_length(v);

    data_t acc = IDENT;
    for (i = 0; i < length; ++i) {
        acc = acc OP data[i];
    }
    *dest = acc;
}
```

改进后速度如下

|   函数   |         选项          | long add | long mul | double add | double mul |
| :------: | :-------------------: | :------: | :------: | :--------: | :--------: |
| combine3 | 移除`get_vec_element` |   4.8    |   6.0    |    6.0     |    7.6     |
| combine4 |   临时变量保存结果    |   0.7    |   1.9    |    1.9     |    3.2     |

可以见到，速度提升非常明显。(这里有一个我非常疑惑的点，理论上long add对应的汇编指令时延下限应该是1，long mul对应下限是3，double add对应是3，double mul对应是5，可是这里的结果似乎更像时延下限分别是1，2，2，3，我查阅了intel的*Haswell*架构汇编指令时延并debug了各种情况都没有发现问题所在，欢迎有兴趣的小伙伴一起来讨论这个问题)。

同样，为什么编译器不会帮我们做这一步的优化呢？这是因为，没有人能保证`dest`和`v`中元素的地址是否一致，换言之，如下两个代码的运行结果是不一样的

```c++
combine2(v, &v->data[2]);
combine3(v, &v->data[2]);
```

当dest的地址就是v中第二个元素地址时，结果显然不同。而编译器不会冒着有可能得到不同答案的风险去做优化。虽然看起来这个函数的目的是累加或者累乘，`combine3(v, &v->data[2]);`的结果似乎更合理，但是编译器不应该去推测写这个程序的程序员写这个函数的目的。因此，我们可以人为的默认`dest`不会是`v`中元素的地址，并加以优化，而编译器则不能对此加以优化。

---

#### 基于现代计算机架构的代码优化

前面介绍了一些常见的编译器不会帮我们做的优化技巧，接下来如果想要进一步优化代码，就需要根据*target machine*的特性进行优化，这一节简单介绍现代intel的cpu的基本原理，作为接下来的优化：循环展开和simd的理论基础。

当c或者c++代码被编译成机器码后，现代cpu会对重复的做下面几个事情：

1.  *fetch* & decode。即读取当前行的机器码，并翻译成对应的操作，如对寄存器*rax*和*rbx*求和，读取或者写入内存地址等。
2. *exec*。在这一步中，cpu每个时钟周期会读入多条可以执行的指令，并分配给对应的*functional units*，*functional units*可以看作cpu的核心劳动力，在*Haswell*架构中，一共有7个*functional units*，每个负责不同的任务。
3. commit。即将上一步的结果写入对应的内存或者寄存器中。

上述只是个非常简单的cpu处理过程。每一步占用多个时钟周期。*Haswell*架构的各种基本运算的参数如下图所示

<div style="text-align:center;"><img src="/assets/opt-1/image-20200409233953078.png" width="100%" height="100%"></div>

首先，*Latency* 代表每条指令需要多少个时钟周期完成。如整数加法*Latency*为1，但是上面说过每条cpu指令至少有3个阶段（实际中有10多个阶段），为什么*Latency*可以做到1呢？这是因为cpu有一个重要的`pipelining`的机制。例如有3个加法运算，他们互相执行顺序可以任意变换，那么在cpu中，当第一条指令执行完*fetch*，开始执行*decode*，第二条指令开始执行*fetch*，以此类推，每条指令距上条指令执行完只差了1个时钟周期，因此*Latency*是1。如果一个代码是完全`pipelined`，就是说这个代码每个时钟周期都可以开始一个新的运算，那么这个代码就无法在时延上进行优化了。

*Issue*是指两条同样的指令开始之间的间隔，例如整数加法，第一条整数加法指令执行1个时钟周期后，即使他还没有完成，下一条整数加法指令就可以继续执行。

*Capacity*是指对于每种指令，一共有多少个打工仔，也就是*functional units*可以执行他。例如有4个整数加法可以同时执行，而整数除法每次只能执行一条指令，也就是说必须要等上一条整数除法指令执行完，才能开始执行下一条整数除法指令，因此，他的时延就是他的执行时间，3-30，而Issue也是他的执行时间。

一种更常见的表达*Issue*和*Capacity*的方式是最大吞吐量，*throughput*，他的定义是*Capacity*/*Issue*，含义是每个时钟周期可以执行多少条这个指令，而他的倒数表示最好情况下每条指令所需的时钟周期。

由此，可以得到这些基本运算的时延下限和吞吐量下限，单位为时钟周期，如下表所示

|       函数       |       选项       | long add | long mul | double add | double mul |
| :--------------: | :--------------: | :------: | :------: | :--------: | :--------: |
|     combine4     | 临时变量保存结果 |   0.7    |   1.9    |    1.9     |    3.2     |
|  Latency bound   |                  |    1     |    3     |     3      |     5      |
| Throughput bound |                  |   0.5    |    1     |     1      |    0.5     |

这里需要注意为什么Throughput bound对于**long add**是0.5而不是0.25，这是因为每次**add**会对两个变量进行内存读取，而一共有2个单元有内存读取的功能，因此每个时钟周期最多只能读取2个元素，因此读取操作限制了吞吐量下限至少为0.5。（这里combine4的结果有点太好，我目前还没理解为什么）

回到combine4的代码，我们可以将机器码转换成数据流的形式，首先combine4的循环对应的汇编代码如下

```asm
data_t = long, OP = *, %eax 保存acc, %rdx保存data+i的地址，%rcx保存data+len的地址
.L25:                                循环
    imull   (%rdx), %eax             data[i] * acc
    addq    $4, %rdx                 data地址递增
    cmpq    %rcx, %rdx               比较边界
    jne .L25                         继续循环
```

指令之间的依赖关系如下图

<div style="text-align:center;"><img src="/assets/opt-1/opt_diag_1.svg"></div>

cpu执行代码时会根据这样的拓扑图进行执行，对于没有拓扑先后顺序的指令，不保证其运行先后顺序。可以分析出，左边加粗的路径做整数乘法，每一步时延为3，总时延为3n，而右边的路径做整数加法，时延为n。因此，左边的加粗线为*关键路径*（*critical path*），因此可以分析出，对于整数乘法，其**CPE**至少为3。（为啥我的实验结果是2）

#### 循环展开

循环展开就是指减少循环总次数，而在每次循环里把原来多次的循环计算展开成序列化的操作，从而减少每次迭代之间的相互依赖性，不多说，直接上代码

```c++
void combine5(vec_ptr v, data_t *dest) {
    long i;
    long length = vec_length(v);
    long limit = length-1;
    data_t *data = v->data;
    data_t acc = IDENT;

    /* Combine 2 elements at a time */
    for(i=0;i<limit;i+=2){
        acc = (acc OP data[i]) OP data[i+1];
    }

    /* Finish any remaining elements */
    for (; i < length; i++) {
        acc = acc OP data[i];
    }
    *dest = acc;
}
```

可以见到，循环现在变成原来的一半，而每次循环计算原先两次对应的操作，这里称作2x1循环展开，第一个2代表每个循环里做原来2倍的操作，1代表只使用1个临时变量，**CPE**如下

|       函数       |       选项       | long add | long mul | double add | double mul |
| :--------------: | :--------------: | :------: | :------: | :--------: | :--------: |
|     combine4     | 临时变量保存结果 |   0.7    |   1.9    |    1.9     |    3.2     |
|     combine5     |   2x1循环展开    |   0.65   |   1.9    |    1.9     |    3.4     |
|  Latency bound   |                  |    1     |    3     |     3      |     5      |
| Throughput bound |                  |   0.5    |    1     |     1      |    0.5     |

可以见到结果并没有变好，为什么呢，继续分析数据流如下图

<div style="text-align:center;"><img src="/assets/opt-1/opt_diag_2.svg"></div>

这条粗的*关键路径*依然是n​个乘法组成，并没有减少。

一个自然的想法是在每次循环里只有一个乘法操作在关键路径上，而将另一个乘法操作放在另外的路径上，自然的想法是，因为这条关键路径主要是`acc`的变化需要序列化，那么可以再用一个临时变量，储存`data[i+1]`的累积结果，代码如下

```c++
void combine6(vec_ptr v, data_t *dest)
{
    long i;
    long length = vec_length(v);
    long limit = length-1;
    data_t *data = v->data;
    data_t acc0 = IDENT;
    data_t acc1 = IDENT;

    /* Combine 2 elements at a time */
    for(i=0;i<limit;i+=2){
        acc0 = acc0 OP data[i];
        acc1 = acc1 OP data[i+1];
    }

    /* Finish any remaining elements */
    for (; i < length; i++) {
        acc0 = acc0 OP data[i];
    }
    *dest = acc0 OP acc1;
}
```

这里使用了`acc0`存储`data[i]`的累积结果，`acc1`存储`data[i+1]`的累积结果，此时，数据流变成如下图

<div style="text-align:center;"><img src="/assets/opt-1/opt_diag_3.svg"></div>

左边的粗线代表`acc0`，中间的代表`acc1`，这样每条粗线代表n/2​次乘法，**CPE**如下

|       函数       |    选项     | long add | long mul | double add | double mul |
| :--------------: | :---------: | :------: | :------: | :--------: | :--------: |
|     combine5     | 2x1循环展开 |   0.65   |   1.9    |    1.9     |    3.4     |
|     combine6     | 2x2循环展开 |   0.65   |   0.9    |    1.0     |    1.5     |
|  Latency bound   |             |    1     |    3     |     3      |     5      |
| Throughput bound |             |   0.5    |    1     |     1      |    0.5     |

可以见到，除了整数加法外别的结果都好了一倍左右，除了整数加法。这是因为循环本身有太多的负载。

另一个对combine5优化的方案是改变运算顺序，先计算`data[i] OP data[i+1]`，这一步因为不是关键路径，可以同时和`acc`的计算并行，从而达到优化，代码如下

```c++
void combine7(vec_ptr v, data_t *dest)
{
    long i;
    long length = vec_length(v);
    long limit = length-1;
    data_t *data = v->data;
    data_t acc = IDENT;

    /* Combine 2 elements at a time */
    for(i=0;i<limit;i+=2){
        acc = acc OP (data[i] OP data[i+1]);
    }

    /* Finish any remaining elements */
    for (; i < length; i++) {
        acc = acc OP data[i];
    }
    *dest = acc;
}
```

对应的数据流如下

<div style="text-align:center;"><img src="/assets/opt-1/opt_diag_4.svg"></div>

可见，这种方法和combine5一样可以使得*关键路径*变为原来的一半，对应的**CPE**为

|       函数       |     选项     | long add | long mul | double add | double mul |
| :--------------: | :----------: | :------: | :------: | :--------: | :--------: |
|     combine5     | 2x1循环展开  |   0.65   |   1.9    |    1.9     |    3.4     |
|     combine6     | 2x2循环展开  |   0.65   |   0.9    |    1.0     |    1.5     |
|     combine7     | 2x1a循环展开 |   0.65   |   2.2    |    0.9     |    1.5     |
|  Latency bound   |              |    1     |    3     |     3      |     5      |
| Throughput bound |              |   0.5    |    1     |     1      |    0.5     |

**double**类型的结果和combine6类似，但是这里**long mul**的结果不符合预期，这里也是我疑惑的一个点，先放这里，想明白了再回来补充。

循环展开并不是展开的越多，结果越好。这是因为存在*Register Spilling*的问题，就是说*x86_64*架构一共有16个整数寄存器，当临时变量的个数超过16个，那么就需要将临时变量储存在内存中，而这种操作显然是非常耗时的。另外因为cpu本身有吞吐量限制，所以循环展开并不是越多越好。

#### SIMD

SIMD即**single instruction multiple data**，就是一种向量化的编程方式，原理是把一个向量存储在超大的向量寄存器里（SSE2为16bytes, AVX为32bytes），然后对这个寄存器进行一次性的操作，在硬件上实现加速。例如，*x86_64*架构中有16个向量寄存器，`%ymm0`-`ymm15`，每个寄存器长度为32bytes，可以存放8个`int`型数据，然后可以对这个寄存器直接进行向量的加减乘除。不多说，上代码

```c++
#define VBYTES 32

/* Number of elements in a vector */
#define VSIZE VBYTES/sizeof(data_t)
/* $end simd_vec_sizes */

/* $begin simd_vec_t */
/* Vector data type */
typedef data_t vec_t __attribute__ ((vector_size(VBYTES)));

char simd_v1_descr[] = "simd_v1: SSE code, 1*VSIZE-way parallelism";
/* $begin simd_combine-c */
void simd_v1_combine(vec_ptr v, data_t *dest)
{
    long i;
    vec_t accum;
    data_t *data = v->data;
    int cnt = vec_length(v);
    data_t result = IDENT;

    /* Initialize all accum entries to IDENT */
    for (i = 0; i < VSIZE; i++) //line:opt:simd:initstart
    	accum[i] = IDENT;  //line:opt:simd:initend

    /* Single step until have memory alignment */
    while ((((size_t) data) % VBYTES) != 0 && cnt) {  //line:opt:simd:startstart
    	result = result OP *data++;
    	cnt--;
    }                              //line:opt:simd:startend

    /* Step through data with VSIZE-way parallelism */
    while (cnt >= VSIZE) {    //line:opt:simd:loopstart
    	vec_t chunk = *((vec_t *) data);
    	accum = accum OP chunk;
    	data += VSIZE;
    	cnt -= VSIZE;
    } //line:opt:simd:loopend

    /* Single-step through remaining elements */
    while (cnt) { //line:opt:simd:loopfinishstart
    	result = result OP *data++;
    	cnt--;
    } //line:opt:simd:loopfinishend

    /* Combine elements of accumulator vector */
    for (i = 0; i < VSIZE; i++) //line:opt:simd:accumfinishstart
    	result = result OP accum[i]; //line:opt:simd:accumfinishend

    /* Store result */
    *dest = result;
}
```

这里用

```c++
typedef data_t vec_t __attribute__ ((vector_size(VBYTES)));
```

声明了一个向量的类型用来存储临时变量，每次迭代时，通过

```c++
vec_t chunk = *((vec_t *) data);
```

将当前数组转换成向量，然后直接通过

```c++
accum = accum OP chunk;
```

进行向量的`OP`运算。

有一点需要注意的是，转换成向量的数组，初始地址必须是32bytes对齐的，这样可以提高读取速度，这里对齐的主要原因是，cpu读取内存的时候都是按照字读取的，也就是说每次读取64位长的内容，如果不对齐的话就需要对64位长的数据进行分割，增加负担。如果想要让`data`数据类型一开始就保证对齐，可以用如下声明

```c++
data_t data[N] __attribute((aligned(32)));
```

括号里为对应的对齐的字节。

另外，intel提供了SSE和AVX的指令集，用法和上述方法类似，不过是定义不同，详见[IntrinsicsGuide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/)。

**CPE**如下所示，另外还提供了将smid进行8x8循环展开的结果。详细代码见最后

|       函数       |     选项     | long add | long mul | double add | double mul |
| :--------------: | :----------: | :------: | :------: | :--------: | :--------: |
|     combine6     | 2x2循环展开  |   0.65   |   0.9    |    1.0     |    1.5     |
|     smid_v1      |     smid     |   0.26   |   1.5    |    0.52    |    0.9     |
|     smid_v8      | smid+8x8展开 |   0.13   |   0.9    |    0.26    |    0.26    |
|  Latency bound   |              |    1     |    3     |     3      |     5      |
| Throughput bound |              |   0.5    |    1     |     1      |    0.5     |

#### 后记

至此，一些基本的线性代码优化已经完成，有几点需要补充说明。

一是代码的正确性，需要保证在优化后代码依然正确，但是很多时候优化所带来的bug是很难被发现的，例如循环展开导致的浮点数计算误差，或者边界条件的欠考虑等，需要进行大量的测试来保障优化的正确性。

另一个是**分支预测**，在遇到判断语句时，cpu不会等到判断表达式的结果出来再去跳转分支，而是直接先运行`if`后面的分支，如果发现这个分支不正确，则取消这个分支上的所有计算，这个代价是很大的，所以在写代码时需要尽量将可能走的分支写在前面。

还有**profile**的问题，众所周知，检查内存泄漏`valgrind --tool=callgrind --simulate-cache=yes program-to-run program-arguments`可以使用

```bash
valgrind --tool=memcheck ./a.out
```

而**profile**代码在linux上可以使用

```bash
gprof ./a.out
```

在mac上我发现clion自带的profiler也是很好用的。

#### 参考文献

[csapp第五章](http://csapp.cs.cmu.edu/)

### 完整代码

```c++
#include <iostream>
#include <chrono>
#include <cstdlib>
#include <ctime>
using namespace std;
#define CLKT CLOCK_THREAD_CPUTIME_ID

#define MHZ 2.2
//#define N 10000000
int N;

#ifdef LONG
typedef long    data_t;
#define TYPENAME "long"
#endif

#ifdef DOUBLE
typedef double data_t;
#define TYPENAME "double"
#endif

#ifdef SUM
#define OP +
#define IDENT 0
#define OPNAME "sum"
#endif 

#ifdef PROD
#define OP *
#define IDENT 1
#define OPNAME "product"
#endif

typedef struct {
    long len;
    data_t *data;
} vec_rec, *vec_ptr;

int get_vec_element(vec_ptr v, long index, data_t *dest) {
    if (index < 0 || index >= v->len) {
        return 0;
    }
    *dest = v->data[index];
    return 1;
}

long vec_length(vec_ptr v) {
    return v->len;
}

char *combine1_desc = "combine1, Maximum use of data abstraction";
void combine1(vec_ptr v, data_t *dest) {
    long i;
    *dest = IDENT;
    for (i = 0; i < vec_length(v); ++i) { 
        data_t val;
        get_vec_element(v, i, &val);
        *dest = *dest OP val;
    }
}

char *combine2_desc = "combine2, Move vec_length out of loop";
void combine2(vec_ptr v, data_t *dest) {
    long i;
    *dest = IDENT;
    long length = vec_length(v);
    for (i = 0; i < length; ++i) { 
        data_t val;
        get_vec_element(v, i, &val);
        *dest = *dest OP val;
    }
}

char *combine3_desc = "combine3, Array reference to data";
void combine3(vec_ptr v, data_t *dest) {
    long i;
    *dest = IDENT;
    data_t *data = v->data;
    long length = vec_length(v);

    for (i = 0; i < length; ++i) { 
        *dest = *dest OP data[i];
    }
}


char *combine4_desc = "combine4, Accumulate in temporary variable";
void combine4(vec_ptr v, data_t *dest) {
    long i;
    *dest = IDENT;
    data_t *data = v->data;
    long length = vec_length(v);

    data_t acc = IDENT;
    for (i = 0; i < length; ++i) { 
        acc = acc OP data[i];
    }
    *dest = acc;
}

/* 2 x 1 loop unrolling */
char *combine5_desc = "combine5, 2 x 1 loop unrolling";
void combine5(vec_ptr v, data_t *dest) {
    long i;
    long length = vec_length(v);
    long limit = length-1;
    data_t *data = v->data;
    data_t acc = IDENT;
    
    /* Combine 2 elements at a time */
    for(i=0;i<limit;i+=2){
        acc = (acc OP data[i]) OP data[i+1];
    }

    /* Finish any remaining elements */
    for (; i < length; i++) {
        acc = acc OP data[i];
    }
    *dest = acc;
}

char *combine6_desc = "combine6, 2 x 2 loop unrolling";
void combine6(vec_ptr v, data_t *dest) {
    long i;
    long length = vec_length(v);
    long limit = length - 1;
    data_t *data = v->data;
    data_t acc0 = IDENT;
    data_t acc1 = IDENT;

    /* Combine 2 elements at a time */
    for (i = 0; i < limit; i += 2) {
        acc0 = acc0
        OP data[i];
        acc1 = acc1
        OP data[i + 1];
    }

    /* Finish any remaining elements */
    for (; i < length; i++) {
        acc0 = acc0
        OP data[i];
    }
    *dest = acc0
    OP acc1;
}

char *combine7_desc = "combine7, 2 x 1 loop unrolling, reassociated";
void combine7(vec_ptr v, data_t *dest) {
    long i;
    long length = vec_length(v);
    long limit = length - 1;
    data_t *data = v->data;
    data_t acc = IDENT;

    /* Combine 2 elements at a time */
    for (i = 0; i < limit; i += 2) {
        acc = acc
        OP(data[i]
        OP
        data[i + 1]);
    }

    /* Finish any remaining elements */
    for (; i < length; i++) {
        acc = acc
        OP data[i];
    }
    *dest = acc;
}

#define VBYTES 32

/* Number of elements in a vector */
#define VSIZE VBYTES/sizeof(data_t)
/* $end simd_vec_sizes */

/* $begin simd_vec_t */
/* Vector data type */
typedef data_t vec_t __attribute__ ((vector_size(VBYTES)));

char simd_v1_descr[] = "simd_v1: SSE code, 1*VSIZE-way parallelism";
/* $begin simd_combine-c */
void simd_v1_combine(vec_ptr v, data_t *dest) {
    long i;
    vec_t accum;
    data_t *data = v->data;
    int cnt = vec_length(v);
    data_t result = IDENT;

    /* Initialize all accum entries to IDENT */
    for (i = 0; i < VSIZE; i++) //line:opt:simd:initstart
        accum[i] = IDENT;  //line:opt:simd:initend

    /* Single step until have memory alignment */
    while ((((size_t) data) % VBYTES) != 0 && cnt) {  //line:opt:simd:startstart
        result = result
        OP * data++;
        cnt--;
    }                              //line:opt:simd:startend

    /* Step through data with VSIZE-way parallelism */
    while (cnt >= VSIZE) {    //line:opt:simd:loopstart
        vec_t chunk = *((vec_t *) data);
        accum = accum
        OP chunk;
        data += VSIZE;
        cnt -= VSIZE;
    } //line:opt:simd:loopend

    /* Single-step through remaining elements */
    while (cnt) { //line:opt:simd:loopfinishstart
        result = result
        OP * data++;
        cnt--;
    } //line:opt:simd:loopfinishend

    /* Combine elements of accumulator vector */
    for (i = 0; i < VSIZE; i++) //line:opt:simd:accumfinishstart
        result = result
    OP accum[i]; //line:opt:simd:accumfinishend

    /* Store result */
    *dest = result;
}

char simd_v8_descr[] = "simd_v8: SSE code, 8*VSIZE-way parallelism";
void simd_v8_combine(vec_ptr v, data_t *dest) {
    long i;
    vec_t accum0, accum1, accum2, accum3, accum4, accum5, accum6, accum7;
    data_t *data = v->data;
    int cnt = vec_length(v);
    data_t result = IDENT;

    /* Initialize accum to IDENT */
    for (i = 0; i < VSIZE; i++)
        accum0[i] = IDENT;
    accum1 = accum0;
    accum2 = accum0;
    accum3 = accum0;
    accum4 = accum0;
    accum5 = accum0;
    accum6 = accum0;
    accum7 = accum0;

    /* Single step until have memory alignment */
    while ((((size_t) data) % VBYTES) != 0 && cnt) {
        result = result
        OP * data++;
        cnt--;
    }                              //line:opt:simd:startend

    while (cnt >= 8 * VSIZE) {
        vec_t chunk0 = *((vec_t *) data);
        vec_t chunk1 = *((vec_t *) (data + VSIZE));
        vec_t chunk2 = *((vec_t *) (data + 2 * VSIZE));
        vec_t chunk3 = *((vec_t *) (data + 3 * VSIZE));
        vec_t chunk4 = *((vec_t *) (data + 4 * VSIZE));
        vec_t chunk5 = *((vec_t *) (data + 5 * VSIZE));
        vec_t chunk6 = *((vec_t *) (data + 6 * VSIZE));
        vec_t chunk7 = *((vec_t *) (data + 7 * VSIZE));
        accum0 = accum0
        OP chunk0;
        accum1 = accum1
        OP chunk1;
        accum2 = accum2
        OP chunk2;
        accum3 = accum3
        OP chunk3;
        accum4 = accum4
        OP chunk4;
        accum5 = accum5
        OP chunk5;
        accum6 = accum6
        OP chunk6;
        accum7 = accum7
        OP chunk7;
        data += 8 * VSIZE;
        cnt -= 8 * VSIZE;
    }
    while (cnt) {
        result = result
        OP * data++;
        cnt--;
    }
    accum0 = (accum0
    OP
    accum1) OP(accum2
    OP
    accum3);
    accum0 = accum0
    OP(accum4
    OP
    accum5) OP(accum6
    OP
    accum7);
    for (i = 0; i < VSIZE; i++)
        result = result
    OP accum0[i];
    *dest = result;
}


template<typename Func>
void profile(Func combine, vec_ptr v, char* str) {
    double CPE_best = 1e10, CPE, chrono_best = 1e10;
    data_t val;
    int i;
    for (i = 0; i < 20; ++i) {
        auto start = chrono::system_clock::now();
        combine(v, &val);
        auto end = chrono::system_clock::now();
        chrono::duration<double> diff = end - start;
        chrono_best = diff.count() / v->len * 2.2 * 1e9 < chrono_best ? diff.count() / v->len * 2.2 * 1e9 : chrono_best;
    }
    cout << str << endl;
    cout << "type:\t\t" << TYPENAME << endl;
    cout << "operation :\t" << OPNAME << endl;
    cout << "chrono CPE:\t\t" << chrono_best << endl;
}
int main() {
    N = rand();
    vec_ptr v = (vec_ptr) malloc(sizeof(vec_rec));
    v->data = (data_t *) calloc(N, sizeof(data_t));
    v->len = N;
    long i;
    for (i = 0; i < N; ++i) {
        v->data[i] = static_cast<data_t>(i % 10);
    }
    profile(combine1, v, combine1_desc);
    profile(combine2, v, combine2_desc);
    profile(combine3, v, combine3_desc);
    profile(combine4, v, combine4_desc);
    profile(combine5, v, combine5_desc);
    profile(combine6, v, combine6_desc);
    profile(combine7, v, combine7_desc);
    profile(simd_v1_combine, v, simd_v1_descr);
    profile(simd_v8_combine, v, simd_v8_descr);
    free(v->data);
    free(v);
}
```

