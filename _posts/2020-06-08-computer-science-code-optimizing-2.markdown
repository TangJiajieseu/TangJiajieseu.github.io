---
layout: post
title:  "[计算机技术] 代码优化2-缓存读写优化"
date:   2020-06-08 15:57:52 +0800
comments: true
categories: [计算机技术, 优化, 循环展开, intel架构]
mathjax: true
---

* TOC
{:toc}

> "Premature optimization is the root of all evil."

## 概述
首先缓存读写优化是什么意思？就是通过改变访问内存肿数据的顺序来提高程序的执行速度。那么为什么需要通过优化内存来提高计算速度呢？那是因为由于摩尔定律，随着晶体管越做越小，计算机的计算能力指数增长，然而，内存的读写速度增长速度却远远比计算机的计算能力低的多，因此，很多时候对于读写频繁的代码，往往bottleneck是内存读写而不是cpu的计算。对么为什么改变访问顺序就能提高速度呢？这是因为**cache**的存在。**cache**就是cpu和内存之间的缓存区。举个简单的例子，明天要交final project，你去图书馆通宵赶报告，需要不断的从书架上借书还书。那你会每次把所有相关的书全部拿到桌子上，直接从桌子上查阅自己需要的部分，查阅完了还回去再去借另一个部分相关的书。如果是cpu是你超频过热的大脑，内存就是整个图书馆的书，那么桌子就是**cache**。**cache**的特点是访问速度快，但是容量小。诶为啥不能搞个速度又快，容量又大的内存呢，这样不就不需要**cache**了？这是因为内存读写的速度最直接的决定因素就是它与cpu的距离。电信号通过光速传播，但是由于这是最底层的结构，一点距离会非常影响时延。在主板上，cpu就是故宫，L1，L2，L3分别是一环二环三环，内存就到五环，硬盘基本就到北23环天津了。离故宫越近，地越珍贵，面积越小，速度也越快。

<div style="text-align:center;"><img src="/assets/opt-2/moore.png" width="100%" height="100%"></div>

## **cache**的工作原理

**cache**之所以有用的最重要特性为：*时间局部性*和*空间局部性*。*时间局部性*是指之前访问过的地址过会儿可能还会再次访问，对应你会反复把同一本书里的知识点拿来看。*空间局部性*是指访问过的内存地址相邻的地址很可能被再次访问，对应你很可能马上回看一本书的下一页。为了充分利用这两个特性，**cache**会每次储存当前读取的地址以及它附近地址的值，当下次再读取这个地址或者相邻地址时就不需要再去内存里找，只需要从**cache**里找就行了，速度快很多。如果读取的顺序不满足这两个特性，那么**cache**就变得毫无意义了。

---

#### Direct-Mapped Cache

<div style="text-align:center;"><img src="/assets/opt-2/direct.png" width="100%" height="100%"></div>

如图是读取或者写入32位内存地址对应的**cache**，有256行**cache line**，每个**cache line**能够储存4个字（1字=4字节），其中第31-12位对应的是Tag，11-2对应的是Index，3-2对应**Block offset**，也就是字的偏置，1-0字节偏置一般情况都是00。

**Tag**:每次读时，会把31-12位的地址与缓存中的tag值进行比较，如果一致，代表读取的值已经在**cache**中了，也就是**cache hit**，直接从cache中读取对应的数据；否则代表需要的值不在**cache**中，也就是**cache miss**，此时需要将当前的数据踢掉，将需要读取的值写入当前的位置。

**Index**:对应内存应该在**cache**中对应的位置，对于256个**cache line**，每个**Index**的值对应一个**cache line**，每个内存地址在**cache**中的位置都由**Index**唯一的确定了。

**Block offset**:对应的字偏置。在_risc_中，读取的内存都是以字为一个单位的，因此，地址的最后两位一定为00，所以1-0的值总是为00。在此例中，每次将数据从内存存入**cache**，还会将它相邻的3个字同时储存到**cache line**中。

另外一个细节是每个**cache line**都有一个**valide bit**来指示这一行是否被初始化过。为了避免可能出现初始化时**Tag**地址是随机的，刚好与读取的内存地址一致，被误判为**cache hit**的情况（后面在多线程中还有别的作用，此处暂且不表）。

#### N-way Set-Associative Cache

<div style="text-align:center;"><img src="/assets/opt-2/nway.jpg" width="100%" height="100%"></div>

**Direct-Map Cache**类似于开地址哈希表，而这个N路的相当于拉链法哈希表，对于如图N=4，每个**Index**有4行**cache line**，每次在**cache**中搜索**tag**时需要比较4次。原理也很简单。

另一个细节是当发生**cache miss**，需要决定踢掉这4个里面的哪一个。可以选用随机法，不过更高明的是**Least Recently Used**(LRU)，就是踢掉最久之前访问过的那个。维护LRU的成本太高，一般直接近似，用一个指针保证不踢掉最近使用过的就可以了。

当N=1的时候，这个方法就是**Direct-Map Cache**，当N=**cache line**时，这个方法就变成**Fully Associative Cache**，不再此处赘述了。

---

当写入数据时，也有两种策略。

**write through**：每次写入更新cache和内存中的值，当写入频繁时，**cache**的作用几乎不存在。

**write back**：每次写入时如果**cache hit**，只更新**cache**中的值，并设置dirty位为1，当踢掉这一行时写入内存，并复位dirty位。如果发生**cache miss**，则写入内存后再保存到**cache**。

## 例子

搞一个小代码来实验一下，求1024\*1024的图像的3\*3大小的均值滤波

1.首先是先遍历列，再遍历行，这种方案完全没有利用**locality**，每次遍历完内层循环开始下一次外层循环都需要把缓存的值全部踢掉再重新储存新的值。

```c++
void boxFilterWH(const Image &in, Image &blury) {
  cout<<"width outer, height inner"<<endl;
  Image blurx(in.width(), in.height());
  for (int x = 1; x < in.width()-1; x++)
    for (int y = 0; y < in.height(); y++) {
      blurx(x, y) = (in(x-1, y) + in(x, y) + in(x+1,y))/3;
    }
  for (int x = 0; x < in.width(); x++)
    for (int y = 1; y < in.height()-1; y++)
      blury(x, y) = (blurx(x,y-1) + blurx(x,y) + blurx(x,y+1))/3;
}
```

2.然后是先遍历行，再遍历列，在计算`blury`的时候，会从头开始读取`blurx`，此时会将`blurx`已有的保存结果踢掉。不过相比于前一种已经好很多了。

```c++
void boxFilterHW(const Image &in, Image &blury) {
  cout<<"height outer, width inner"<<endl;
  Image blurx(in.width(), in.height());
  for (int y = 0; y < in.height(); y++) {
    for (int x = 1; x < in.width()-1; x++)
      blurx(x, y) = (in(x-1, y) + in(x, y) + in(x+1,y))/3;
    }
  for (int y = 1; y < in.height()-1; y++)
    for (int x = 0; x < in.width(); x++)
      blury(x, y) = (blurx(x,y-1) + blurx(x,y) + blurx(x,y+1))/3;
}
```

3.接着是对每个`blury`的元素分别计算所有的需要值，完全利用了**locality**的特性，但是问题是无法并行化，但目前我们还没有考虑并行的问题。

```c++
void boxFilterL(const Image &in, Image &blury) {
  cout<<"Maximum use of Locality"<<endl;
  Image blurx(in.width(), in.height());
  for (int y = 2; y < in.height(); y++)  {
    for (int x = 1; x < in.width()-1; x++) {
        blurx(x, y) = (in(x-1, y) + in(x, y) + in(x+1,y))/3;
        blury(x, y - 1) = (blurx(x,y - 2) + blurx(x,y - 1) + blurx(x,y))/3;
    }
  }
}
```

4.最后是`halide`展现的一个版本，把图像分成很多小的tile，然后对每个tile求得所有需要的值，并行化容易得多。

```c++
void boxFilterT(const Image &in, Image &blury) {
  int yTileSize = CACHE_LINE * 2, xTileSize = LINE_NUMBER / 2 / 2 ;
  cout<<"tile computation, tile size: ["<<yTileSize<<" , "<< xTileSize << "]"<<endl;
  Image blurx(in.width(), in.height());
  for (int yTile = 0; yTile < in.height() ; yTile += yTileSize) {
    for (int xTile = 0; xTile < in.width(); xTile += xTileSize) {
      for (int y = yTile - 1; y < yTile + yTileSize + 1; y++) {
        if (y < 0 || y >= in.height()) continue;
        for (int x = xTile; x < xTile + xTileSize; x++) {
            if (x < 1 || x >= in.width() - 1) continue;
            blurx(x, y) = (in(x-1, y) + in(x, y) + in(x+1,y))/3;
        }
      }
      for (int y = yTile; y < yTile + yTileSize; y++) {
        if (y < 1 || y >= in.height()-1) continue;
        for (int x = xTile; x < xTile + xTileSize; x++) {
            if (x < 0 || x >= in.width()) continue;
            blury(x, y) = (blurx(x,y-1) + blurx(x,y) + blurx(x,y+1))/3;
        }
      }
    }
  }
}
```

速度上来说，第一个最慢，第二个由于**locality**虽然有多余操作，但是对整体计算影响没那么大，因此速度略慢于第三个和第四个，但是速度差异没有特别明显。


## 参考文献

[cs61c](https://inst.eecs.berkeley.edu//~cs61c/sp15/)
[halide](https://halide-lang.org/)

## 完整代码
```c++
#include <iostream>
#include <vector>
#define WORD        4
#define CACHE       (32768 / WORD)
#define CACHE_LINE  (64 / WORD)
#define LINE_NUMBER (CACHE / CACHE_LINE)
using namespace std;
class Image {
public:
  Image(int w, int h, uint16_t color=0) : w(w), h(h) {
    framebuffer = vector<vector<uint16_t>>( h, vector<uint16_t>(w, color));
  }
  uint16_t& operator()(int x, int y) {
    return framebuffer[y][x];
  }
  const uint16_t& operator()(int x, int y) const  {
    return framebuffer[y][x];
  }
  uint16_t* ptr(int x, int y) {
    return &framebuffer[y][x];
  }
  const uint16_t* ptr(int x, int y) const {
    return &framebuffer[y][x];
  }
  int width() const {return w;}
  int height() const {return h;}
private:
  int w, h;
  vector<vector<uint16_t>> framebuffer;  
};

void boxFilterWH(const Image &in, Image &blury) {
  cout<<"width outer, height inner"<<endl;
  Image blurx(in.width(), in.height());
  for (int x = 1; x < in.width()-1; x++) 
    for (int y = 0; y < in.height(); y++) {
      blurx(x, y) = (in(x-1, y) + in(x, y) + in(x+1,y))/3;
    }
  for (int x = 0; x < in.width(); x++)
    for (int y = 1; y < in.height()-1; y++) 
      blury(x, y) = (blurx(x,y-1) + blurx(x,y) + blurx(x,y+1))/3;
}
void boxFilterHW(const Image &in, Image &blury) {
  cout<<"height outer, width inner"<<endl;
  Image blurx(in.width(), in.height());
  for (int y = 0; y < in.height(); y++) {
    for (int x = 1; x < in.width()-1; x++) 
      blurx(x, y) = (in(x-1, y) + in(x, y) + in(x+1,y))/3;
    }
  for (int y = 1; y < in.height()-1; y++) 
    for (int x = 0; x < in.width(); x++)
      blury(x, y) = (blurx(x,y-1) + blurx(x,y) + blurx(x,y+1))/3;
}
void boxFilterL(const Image &in, Image &blury) {
  cout<<"Maximum use of Locality"<<endl;
  Image blurx(in.width(), in.height());
  for (int y = 2; y < in.height(); y++)  {
    for (int x = 1; x < in.width()-1; x++) {
        blurx(x, y) = (in(x-1, y) + in(x, y) + in(x+1,y))/3;
        blury(x, y - 1) = (blurx(x,y - 2) + blurx(x,y - 1) + blurx(x,y))/3;
    }
  }
}
void boxFilterT(const Image &in, Image &blury) {
  int yTileSize = CACHE_LINE * 2, xTileSize = LINE_NUMBER / 2 / 2 ;
  cout<<"tile computation, tile size: ["<<yTileSize<<" , "<< xTileSize << "]"<<endl;
  Image blurx(in.width(), in.height());
  for (int yTile = 0; yTile < in.height() ; yTile += yTileSize) {
    for (int xTile = 0; xTile < in.width(); xTile += xTileSize) {
      for (int y = yTile - 1; y < yTile + yTileSize + 1; y++) {
        if (y < 0 || y >= in.height()) continue;
        for (int x = xTile; x < xTile + xTileSize; x++) {
            if (x < 1 || x >= in.width() - 1) continue;
            blurx(x, y) = (in(x-1, y) + in(x, y) + in(x+1,y))/3;
        }
      }
      for (int y = yTile; y < yTile + yTileSize; y++) {
        if (y < 1 || y >= in.height()-1) continue;
        for (int x = xTile; x < xTile + xTileSize; x++) {
            if (x < 0 || x >= in.width()) continue;
            blury(x, y) = (blurx(x,y-1) + blurx(x,y) + blurx(x,y+1))/3;
        }
      }
    }
  }
}
template<typename Func>
void profile(Func f) {
  int w = 1024, h = 1024;
  Image in(w, h, 0x7777), blury(w, h);
  auto t = chrono::system_clock::now();
  f(in, blury);
  chrono::duration<float> d = chrono::system_clock::now() - t;
  cout<<"duration: "<<d.count()<<endl;
}

int main() {
 profile(boxFilterHW); 
 profile(boxFilterL); 
 profile(boxFilterT); 
 profile(boxFilterWH); 
}
```

## 后记
缓存优化本质上是一个求最优调度方式的问题，一些常见的图形图像加速编程语言如halide, taichi本质上就是把算法和调度分离，在处理规则稠密的数据时，本质上就是在固定算法的情况下找到最优的调度策略实现最少的**cache miss**，从而加速算法。

**locality**在并行中也有非常大的重要性，包括缓存一致性，不同线程之间的内存访问冲突，false sharing等，再下一节会具体介绍。
