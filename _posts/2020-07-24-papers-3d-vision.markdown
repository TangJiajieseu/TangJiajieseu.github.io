---
layout: post
title:  "[paper reading] 3D视觉论文赏析"
date:   2020-07-24 23:29:00 +0800
comments: true
categories: [计算机技术, 机器学习，3D视觉]
mathjax: true
---

* TOC
{:toc}

### 前言
记录一些比较好的3d视觉相关的paper，工作比较忙，所以论文只介绍核心思想和大概内容。

### slam

### 3D detection/segmentation/feature extraction


### stereo/multiview
#### [Deep Stereo using Adaptive Thin Volume Representation with Uncertainty Awareness](https://arxiv.org/abs/1911.12012)
背景是多目估深度做三维重建。

传统方法是每个角度的图片$$ \textbf{I}_i $$ 通过cnn提特征之后得到$$F_i$$，对于每个深度$$d$$，可以通过相机的内参和外参将$$$F_i$$的每个像素点投影回reference image，得到对应的像素位置，由此，空间中的每一个点，都可以得到对应的特征，理论上来说，如果这个点是物体的表面，那么各个视角的特征差异很小，反之就会很大。由此，只要在空间中**均匀**采样$$d$$，就可以把空间从前到后均匀分成$$d$$份slice，每个slice可以看作对应深度的平面，那么就可以得到一个空间三维tensor，tensor的每个值表征这个位置可能是物体表面的概率，这个三维立方体就是cost volume。cost volume中的值可以通过这个点对应像素位置的特征求方差得到。最终把这个cost volume通过3D CNN来预测最终的sdf。

传统的做法问题是，$$F_i$$尺寸越大越精确，但是计算量越大，而且空间中绝大部分位置都不是表面，所以这个cost volume其实很稀疏。因此就有如下的改进。

<div style="text-align:center;"><img src="/assets/3dv/deep-stereo-AVP.png" width="100%" height="100%"></div>

直接看图，1st stage操作就是传统的做法，得到一个粗略的depth，depth的公式为

$$\hat{\textbf{L}}_{k}(x) = \sum^{D_k}_{j=1}{\textbf{L}_{k, j}(x) \cdot \textbf{P}_{k,j}(x) }$$

其中$$\hat{\textbf{L}}_{k}(x)$$表示像素位置$$x$$在第$$k$$个阶段对应的深度，它等于从$$x$$发一条射线，经过所有的深度采样点$$\textbf{L}_{k, j}(x)$$乘这个点对应的概率$$\textbf{P}_{k,j}(x)$$，本质上就是一个加权求和。

得到1st stage的结果之后，可以通过预测的depth map求每个像素对应的标准差：

$$\hat{\sigma}_k(x) = \sqrt{\hat{\textbf{V}}_{k}(x)} = \sqrt{\sum^{D_k}_{j=1}{\textbf{P}_{k, j}(x) \cdot ( \textbf{L}_{k,j}(x) - \hat{\textbf{L}}_{k}(x))^2 }}$$

这部分比较好理解，就是标准的求方差公式

得到第一次估计的值$$\hat{\textbf{L}}_{k}(x)$$（可以看作是均值）和标准差，那2nd stage在对深度进行采样时范围就不需要那么大，只需要在均值附近区域进行均匀采样就可以缩小采样范围同事减小运算量，采样范围是

$$\textbf{C}_{k}(x) = [\hat{\textbf{L}}_{k}(x) -\lambda \hat{\sigma}_k(x) , \hat{\textbf{L}}_{k}(x) + \lambda \hat{\sigma}_k(x)] $$

其中$$\lambda$$是个超参。剩下的就是重复操作，得到最终结果了，比较容易。

可以看到每个stage对应的3D CNN的通道数都是越来越小的，因为他们只需要预测距离上一次的均值偏差有多少。

