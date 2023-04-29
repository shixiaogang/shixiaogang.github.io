---
title: "Lossy Counting频数估计算法"
date: 2023-03-20
categories:
  - "概要算法"
  - "频数估计"
tags:
  - "数据流"
  - "概要算法"
  - "频数估计"
toc: true
---



LossyCounting是R. Motwani和G. S. Manku在[1]中提出的另一个频数估计算法（该论文中的另一个算法为 [Sticky Sampling算法](../sticky-sampling)）。虽然LossyCounting算法在[Misra-Gries算法](../misra-gries)提出之后20年提出，但LossyCounting算法在估计误差上和Misra-Gries算法是一样的，在空间复杂度和计算复杂度上还不如Misra-Gries算法。

LossyCounting算法的对数据项的频数的估计可以满足$0 \le f - \hat{f} \le \epsilon n$，其中$f$为真实频数，$\hat{f}$为估计频数，$n$为所有频数之和；所需的记录数为$\frac{1}{\epsilon}\log(\epsilon n)$。LossyCounting算法也可以对数据流中的频繁项进行估计，并对给定的阈值$s\in(0,1)$满足

* 所有真实频数超过$sn$的数据项都能够被返回。
* 所有真实频数少于$(s-\epsilon)n$的数据项都不会被返回。



## 算法实现

LossyCounting算法将数据流划分成一个个窗口。每个窗口中包含了$\lceil 1/\epsilon \rceil$个元素。每个窗口从1开始被编上号，当前窗口的编号即为$b_{current}$。如果当前已经处理的元素总数为$n$，则$b_{current}$的值为$\left\lfloor \frac{n}{\lceil 1/\epsilon \rceil} \right\rfloor \le n\epsilon$。

LossyCounting算法维护了一组记录$(e, f, \Delta)$，其中$e$为数据流中的数据项，$f$为对$e$的频数的估计，而$\Delta$则是$f$的最大可能的估计误差，也是这个元素在之前$b_{current}-1$个窗口中最多可能出现的次数。我们将这组记录记为$\mathcal{S}$。

当一个新元素$e$到达时，我们首先检查$e$是否存在于$\mathcal{S}$中。如果$e$已经存在于$\mathcal{S}$中，那么我们直接将其对应的$f$加1。如果$e$没有在$\mathcal{S}$中，那么我们在$\mathcal{S}$中添加一个新记录$(e, 1, b_{current} - 1)$。每当我们处理完一个窗口后，我们就对$\mathcal{S}$进行清理。对于一个记录$(e, f, \Delta)$，如果$f + \Delta \le b_{current}$，那么我们就将其从$\mathcal{S}$中删除。

当我们需要查询某个数据项$e$的频数时，如果$e$在$\mathcal{S}$中，我们返回$f$作为其频数的估计；如果$e$不在$\mathcal{S}$中，则返回0作为频数的估计。而当我们需要查询数据流中出现频数超过$sn$的数据项时，我们返回$\mathcal{S}$中所有$f \ge (s-\epsilon)n$的数据项。

## 算法分析

**引理1**. 当一个记录$(e, f, \Delta)$被删除时，一定有$f_e \le b_{current}$，其中$f_e$为$e$的真实频数。

**证明**. 我们通过归纳法证明。

当$b_{current}=1$时，即在第一个窗口时，所有记录的$f$都是和其对应真实频数相同，并且$\Delta$都为0。如果某个记录$(e, f, \Delta)$在第一个窗口结束时被删除，那么一定有$f_e \le b_{current}$。

我们假设$b_{current} < k$时，结论成立。

如果当$b_{current} = k$时，记录$(e, f, \Delta)$被删除。注意到，我们在插入记录时会使用$b_{current}-1$作为$\Delta$的值，并且在之后都不会修改$\Delta$。因此记录$(e, f, \Delta)$一定是在第$\Delta + 1$个窗口中被插入到$\mathcal{S}$中的。

而这个记录可能之前也存在于$\mathcal{S}$中，并在某个窗口$r < k$中从$\mathcal{S}$中删除了。根据我们前面的归纳假设，当这个记录在窗口$r$中被删除时，有$f_e' \le r \le \Delta$，其中$f_e'$是数据项$e$在窗口r中被删除时的真实频数。

由于自从窗口$\Delta +1$插入到$\mathcal{S}$之后，$e$的每次出现都会记录到了对应的$f$中，因此有$f_e = f_e' + f \le \Delta + f$。

注意到，我们在窗口$b_{current}$中只会删除$f + \Delta \le b_{current}$的记录，因此有$f_e \le b_{current}$。

结论在$b_{currnet} = k$时也成立，命题得证。



**定理1**. 对于任意一个数据项$e$，LossyCounting算法对其频数的估计$\hat{f}$满足

$$0 \le f - \hat{f} \le \epsilon n$$

**证明**. 如果$e$不在$\mathcal{S}$中，则$\hat{f} = 0$。则其一定在之前某个窗口$b' \le b_{current}$中被删除。根据*引理1*可知

$$0 \le f \le b' \le b_{current} \le \epsilon n$$

如果$e$在$\mathcal{S}$中，并且$\Delta = 0$，则说明$e$在第一个窗口就被添加到$\mathcal{S}$中，$f = \hat{f}$。

如果$e$在$\mathcal{S}$中，并且$\Delta \ge 1$，则说明$e$在窗口$\Delta +1$中被添加到$\mathcal{S}$中。$e$可能在之前的某个窗口$b'$从$\mathcal{S}$中被删除了。根据*引理1*可知，在删除时$e$的真实频数不超过$b'$，从而

$$0 \le f - \hat{f} \le b' \le \Delta \le b_{current} \le \epsilon n$$

综上，命题得证。



**定理2**. LossyCounting算法所保存的记录数最多为$\frac{1}{\epsilon}\log(\epsilon n)$。

**证明**.  令$B = b_{current}$为当前窗口的编号。对于$i \in [1, B]$，令$d_i$表示$\mathcal{S}$中在窗口$i$插入到$\mathcal{S}$中的记录数目，即$\Delta = i - 1$的记录数目。

由于我们在每个窗口结束时都会清理$\mathcal{S}$中的记录。因此，对于$\Delta = B-i$的记录，其数据项在窗口$B-i+1$到窗口$B$之间一定至少出现了$i$次；否则其就会在这期间从$\mathcal{S}$中删除。

令$w = \lceil 1/\epsilon \rceil$为每个窗口的大小，则我们有

$$\sum_{i = 1}^j i \cdot d_i \le j \cdot w, j = 1, 2, ..., B$$

我们根据归纳法来证明

$$\sum_{i = 1}^j d_i \le \sum_{i = 1}^j \frac{w}{i}, j = 1, 2, ..., B$$

当$j = 1$时，上式显然成立。

假设当$j <k$时，上式成立。

由于
$$
\begin{aligned}
& k\sum_{i = 1}^k d_i \\\\
=& \sum_{i = 1}^k i \cdot d_i + \sum_{i = 1}^1 d_i + \sum_{i=1}^2 d_i + ... + \sum_{i = 1}^{k-1} d_i \\\\
\le & kw + \sum_{i=1}^{k-1} \frac{(k-i)w}{i} \\\\
=& k\sum_{i=1}^k \frac{w}{i}
\end{aligned}
$$
因此，当$j = k$时，$\sum_{i = 1}^j d_i \le \sum_{i = 1}^j \frac{w}{i}$，结论成立。

由于$|\mathcal{S}| = \sum_{i = 1}^B d_i$，因此

$$|\mathcal{S}| \le \sum_{i=1}^B \frac{w}{i} \le w \log B \le \frac{1}{\epsilon}\log(\epsilon n)$$



## 参考文献

[1] G. S. Manku, R. Motwani. *Approximate Frequency Counts over Data Streams*. In VLDB 2002, pp. 346-357.