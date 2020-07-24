---
layout: post
title:  "[paper reading] 2d相关论文赏析"
date:   2020-07-25 00:32:00 +0800
comments: true
categories: [计算机技术, 机器学习，2d检测]
mathjax: true
---

* TOC
{:toc}

### 前言
记录一些比较好的2d相关的paper，工作比较忙，所以论文只介绍核心思想和大概内容。

### 2d detection

[https://zhuanlan.zhihu.com/p/163044323](https://zhuanlan.zhihu.com/p/163044323)
一篇eccv的oral，通过融入边界点的信息，轻松涨点
<div style="text-align:center;"><img src="/assets/2d/bordernet.png" width="100%" height="100%"></div>
这篇基于fcos。fcos加了一个center head，判断点是center的概率，把这个概率乘以对应位置分类概率得到最终提供给nms的概率。而这篇提到可以增加边界的信息。

具体做法是先预测一个框，然后把每个点的预测框的边对应的通道做pooling采样得到采样点最大的值（这里他说的不太清楚，到底是pooling得到Cx1x1的特征还是采样一个点，用那个点的特征，这个采样最大值又应该怎么计算？），对每个点就得到5C个通道的特征，其中4C来自边的pooling，还有一个C来自自身，然后用这个特征再去做cls和reg。

其实这个方法有点讨巧的点在于他既不算严格意义的2d检测，但是又的确两次预测了框，挺有意思的。另外这个方法在框里均匀采样就变成roialign，采样一个点就变成fcos，对每个边加权求和就变成另一个paper（不记得叫啥了），而如果取9个格子每个格子对应一个通道就变成了R-FCN。

[End-to-End Object Detection with Transformers](https://arxiv.org/pdf/2005.12872.pdf)
一篇eccv的oral，用transformer做detection，直接上图


<div style="text-align:center;"><img src="/assets/2d/detr.png" width="100%" height="100%"></div>

思想很简单，2d feature加上与图像无关的位置信息，转成WHxNxC的格式输入transformer的encoder部分，decoder部分输入为0，并加上位置信息，也就是nn.embedding的权重，然后输出多个head，每个head分别对应cls和box，得到之后通过匈牙利算法算box和ground truth之间的匹配，求loss。

非常想记录这篇文章，因为在此之前我在3d上想过用transformer来做检测，而且3d上的点云没有顺序，所以作为输入，对位置信息的要求远远没有2d的image来的高，甚至最终我也采用了匈牙利算法构造了一个比较复杂的损失函数来算loss，与这个文章比较大的区别在2点。1.我没有像文章中那样频繁的使用位置信息，2.decoder输入不是0，而是上一个的box信息通过embedding，理论上来说这样更合理，我告诉了网络前面的box在什么地方，网络告诉我还有哪些框。但是一方面实验结果比较糟糕，另一方面被mentor狂喷没有意义不合理啥的，心态有点爆炸，所以最终并没有成功。看到这篇paper出来，心里非常复杂，不再赘述了，不过对我的启示是永远永远永远要相信自己。

