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

#### [PointPillars](https://arxiv.org/pdf/1812.05784.pdf)

<div style="text-align:center;"><img src="/assets/3dv/pointpillar.png" width="100%" height="100%"></div>

**PointPoint**是一篇优秀的高效3D检测文章。算法流程如下：

1. 对输入的点云，取如最左边所示的$$B=H\times W$$个柱体，随机采样$$P$$个包含点云的柱体，对每个柱体中随机采样$$N$$个点，每个点的特征为$$D=9$$维，分别是$$\{ x, y, z, r,x_c, y_c ,z_c,x_p, y_p \}$$，其中$$x,y,z$$为点的原始坐标，$$r$$为反射系数，下标$$c$$对应每个点对这个柱体中所有点的平均值的欧式距离，下标$$p$$代表每个点与平均点的坐标偏移。就得到$$(D, P,N)$$的对点云的紧凑表达。
2. 然后对$$D$$这一维通过MLP，将其扩展为$$C$$维，得到$$(C,P,N)$$的特征，再对$$N$$这一维通过max pooling，得到$$(C,P)$$的张量。再把$$P$$个柱体映射回原来对应的$$H\times W$$坐标上，得到$$(C,H,W)$$的稀疏伪图像表达。
3. 通过SSD，得到三维的bounding box。

#### [PVRCNN](https://arxiv.org/pdf/1912.13192.pdf)

<div style="text-align:center;"><img src="/assets/3dv/pvrcnn.png" width="100%" height="100%"></div>

kitti上的(前)sota，上半个分支就是3D稀疏卷积之后把z维和特征维合并得到bev上的特征，然后出rpn；得到rpn后通过roi-grid pooling得到每个proposal的特征，再预测cls和reg。

主要的创新来自下半个分支，主要包括：

**关键点特征提取**:对点云做fps采样，得到key points，然后在每个卷积的特征图上找到每个特征点的位置，把每个特征图上的这个特征点领域的特征做一个SA层的操作，得到这个key point的一维特征，最后还会把bev上的特征做一个双线性插值，全部concat起来得到最终这个特征点的特征。

**PKW**:特征点可能是在前景内的也可能是背景，因此对特征点预测一个前背景的标签，再把预测结果乘以特征，有点像attention的操作，前景的贡献较大，背景的贡献则较小，如下图所示。

<div style="text-align:center;"><img src="/assets/3dv/pkw.png" width="100%" height="100%"></div>

**RoI-grid Pooling**: 对每个proposal，均匀地取三维的6x6x6的grid点，对每个点，取其领域的key points，再做一个SA层的操作，得到每个grid点的特征。然后把这216个特征向量化后通过mlp，预测得到输出。

<div style="text-align:center;"><img src="/assets/3dv/grid-roi.png" width="100%" height="100%"></div>

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

