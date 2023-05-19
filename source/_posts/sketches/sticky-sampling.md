---
title: "Sticky Sampling频数估计算法"
date: 2023-03-12
categories:
  - "概要算法"
  - "频数估计"
tags:
  - "数据流"
  - "概要算法"
  - "频数估计"
toc: true
---



StickySampling是R. Motwani和G. S. Manku在2002年提出的一种基于采样的频繁项估计算法 [1]。在原始论文中，作者号称它能够对数据项的频数以超过$1-\delta$的概率提供相对误差为$\epsilon$的估计，并返回数据流中所有频率超过给阈值的所有数据项，其所需的空间复杂度为$\mathrm{O} \left( \log \frac{1}{\epsilon \delta} \right)$。**但这个结论的证明可能存在问题**。

## 算法实现

在StickySampling算法中，我们维护了一组记录$(e, f)$，其中$e$为数据流中的数据项，而$f$是对$e$频数的估计。我们将这组记录记为$\mathcal{S}$。

每当有一个新元素$e$到达时，如果其已经存在于$\mathcal{S}$中，我们将其对应的频数$f_e$增加1；否则我们以采样率$1/r$对元素进行采样。如果$e$被采样，我们就在$\mathcal{S}$中插入一个新记录$(e, 1)$。和[MorrisCounter](../morris_counter)算法类似，StickySampling也对采样率进行动态调整。最开始的$2t$个元素，采样率为1；而对之后的$2t$个元素，$r$被设置为2，即以采样率$1/2$进行采样；再之后的$4t$个元素的采样率为$1/4$，以此类推。在这里，$t$是一个由误差概率$\delta$和$\epsilon$决定的一个参数。

每当采样率进行调整的时候，我们也会扫描$\mathcal{S}$中的元素进行清理。对于每个$\mathcal{S}$中的记录$(e, f)$，我们投掷一个无偏的硬币。如果硬币反面朝上时，我们就将这个记录的频数$f$减1。如果$f$变为0，那么就将这个记录从$\mathcal{S}$中删除。我们重复这个投掷过程直到硬币正面朝上。*在原始论文中，只提到了投掷使用的硬币是无偏的，而没有对反面朝上的概率进行明确。从后续的证明来看，硬币反面朝上的概率应为$(1-1/r)$。*

当用户需要查询数据流中出现频率超过$s$的数据项时，我们就输出$\mathcal{S}$中$f \ge (s - \epsilon) N$的数据项即可。

## 算法分析

下面是原始论文中的证明。

> 我们按照采样率对数据流划分成一组窗口。第一个窗口包含$2t$个元素，而之后的窗口中都包含了$rt$个元素。假设我们当前窗口的采样率为$1/r$，那么之前窗口中的元素总数为$2t + 2t + 4t + ... + (r/2)t = rt$。我们用$N$表示当前已经处理的元素总数，则$N = rt + rt'\ge rt$，其中$t' \in [0, 1)$。从而$1/r \ge t/N$。
>
> 对于一次衰减操作，我们减去的频数和连续投掷硬币为反面的次数一样，因此这个值服从几何分布，其超过$\epsilon N$的概率不超过$(1 - 1/r)^{\epsilon N}$。由于$1/r \ge t/N$，则这个衰减值超过$\epsilon N$的概率不超过$(1 - t/N)^{\epsilon N}$，进而不超过$e^{-\epsilon t}$。
>
> 在整个数据流中，频数至少为$sN(s \in [0, 1])$的数据项最多只有$1/s$个。这些数据项中任意一个减少的频数超过$\epsilon N$的概率最多只有$e^{-\epsilon t}/s$。令$t \ge \frac{1}{\epsilon}\log \left( \frac{1}{s \delta} \right)$，则这些数据项有有任意一个减少的频数超过$\epsilon N$的概率不超过$\delta$。



在上面的证明过程中，存在以下问题

1. 对于频数估计的误差，来自于两个方面。一个是当数据项还未在$\mathcal{S}$由于采样导致的频数减少；以及在每次调整采样率时的衰减操作。但上面的证明过程并没有分别考虑这两方面的影响。
2. 一个数据项可能经历多次衰减操作，并且在每次衰减操作时，$r$和$N$的值是不同的。但上面的证明过程并没有考虑多次衰减的影响。

## 参考文献

[1] G. S. Manku, R. Motwani. *Approximate Frequency Counts over Data Streams*. In VLDB 2002, pp. 346-357.