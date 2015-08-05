---
layout: post
title: 基数估计
author: 灵元 
---
 
##1 
 基数估计，集合归属检测，topk
 
## 2 算法

### 2.1 Bitmap算法
xx
 

### 2.2 Linear probabilitstic 基数
该算法是上述bitmap算法的概率化，与上述算法相比，不要求元素到bitmap中位的一一映射，而是依赖于一个哈希函数。哈希函数将产生冲突，这就导致了结果的概率性。

1. 构造一个大小为B的bitmap，初始化为0
2. 选择一个哈希函数h，将流中的元素映射为{1,...,B}
3. 对流中的每一个元素 x，计算h(x),将相应的bitmap的位置1
4. 最后计算bitmap中1的数目，设为c
5. 输出最后的基数估计值：$B\ln\frac{B}{B-c}$

分析：

设D为基数，那么bitmap中某个bit为0的概率为$(1-1/B)^D\approx e^{-D/B}$

因而总体0的个数的期望$B-c=B e^{-D/B}$

最终得结果：$D=B\ln\frac{B}{B-c}$

### 2.3 HyperLogLog

算法：

1. 给定一个参数$p$，令$m=2^{p}$,构造一个m个元素的数组M，初始化为0
2. 选择一个哈希函数，将元素映射为一个均匀分布的0-1串（如64位）
3. 对于每一个元素的哈希值$v$，用前$p$位定位数组M的位置，设为j
4. 令$M[j]=max(M[j],z)$,其中，$z$等于$v$的前导0的个数加上1
5. 结果为：$E = 2^{2p} \left(2^p \int_{0}^{\infty} \left( \log_2 \left( \frac{2 + u}{1 + u} \right) \right)^{2^p}{\rm{d}}u\right)^{-1} \left( \sum_{i=0}^{2^p - 1} 2^{-M[i]} \right)^{-1}$