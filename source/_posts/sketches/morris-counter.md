---
title: "Morris频数估计算法"
date: 2023-03-11
categories:
  - "概要算法"
  - "频数估计"
tags:
  - "数据流"
  - "概要算法"
  - "频数估计"
toc: true
---



MorrisCounter算法是R. Morris于1978年提出的一种用于估计频数的算法 [1]. 当时Morris需要编写一段代码来对大量事件进行计数，但是他能使用的只有一个8位的计数器。为了能在有限的存储空间内完成任务，他发明了MorrisCounter算法，能够使用 $ O(\log \log N + \log 1/ \epsilon + \log 1 / \delta ) $个比特，对频数进行估计，并且保证估计频数$\hat{f}$和真实频数$f$之间满足：

$$ Pr[|\hat{f} - f| \le \epsilon f] \ge 1 - \delta $$



## 算法实现

对计数进行估计的一个简单思想就是，*我们不必严格记录下每次看到的新事件*。我们可以以一定的概率记录新事件。每当我们看到一个新事件，我们就抛一枚硬币。如果正面朝上，我们就增加计数；否则我们就忽略它。这种基于多次抛硬币的结果可以通过一个二项式分布来描述。通过二项式分布的标准偏差，我们就可以对估计误差进行分析。

这种抛硬币的方式虽然可以减少计数时所需的空间，但其所需的空间仍然和计数呈线性关系。为了能够进一步获得更好的空间复杂度，MorrisCounter算法对抛硬币的概率进行动态调整，随着计数的增加而逐步降低硬币朝上的概率：当第一次看到事件时，更新的频率为1，下一次为1/2，然后为1/4，以此类推。

在MorrisCounter的实现中，我们使用$c$来保存对当前计数的估计，并使用参数$b \in (1, 2]$来作为是否计数的参数。

MorrisCounter的更新操作实现如下

{% listing title:MorrisCounter更新操作 %}
randomly pick $y$ from $[0, 1]$
if $y < b^{-c}$, then

$~~~~c \gets c + 1$

end if

{% endlisting %}

MorrisCounter的查询操作实现如下

{% listing title:MorrisCounter查询操作 %}

return $(b^c - 1)/(b-1)$

{% endlisting %}

如果两个MorrisCounter的参数$b$是相同的，那么我们也可以将这两个MorrisCounter进行合并。当合并时，我们选择较大的计数值作为基础，将较小的计数值按照更新操作的方式累加到结果上。

{% listing title:MorrisCounter查询操作 %}

$\alpha = \min(c_a, c_b)$, $\beta = \min(c_a, c_b)$

for $j$ in $[0, \alpha]$ 

$~~~~$ randomly pick $y$ from $[0, 1]

$~~~~$ if $y < b^{j-\beta}$

$~~~~~~~~$ $\beta \gets \beta + 1$ 
$~~~~$ end if 
end for
return $\beta$

{% endlisting %}

## 算法分析

令$C_n$表示经过$n$次更新操作之后$c$的值，$X_n = (b^{C_n} - 1)/(b-1)$为经过$n$次更新操作之后对真实频数的估计值。

**引理1**
$$
E[X_n] = n
$$
**证明**.
$$
\begin{aligned}
E[X_n] &= \sum_c Pr[C_n = c] \frac{b^c - 1}{b - 1} \\\\
&= \sum_c (Pr[C_n = c - 1]b^c + Pr[C_{n-1} = c](1 - b^{-c}))\frac{b^c - 1}{b - 1} \\\\
&= \sum_c Pr[C_{n-1} = c] \left( b^{-c} \frac{b^{c+1} - 1}{b-1} + (1 - b^{-c})\frac{b^c-1}{b-1}\right) \\\\
&= \sum_c Pr[C_{n-1} = c] \frac{b^c-1}{b-1} + 1 \\\\
&= E[X_{n-1}] + 1
\end{aligned}
$$


**引理2**.
$$ var[X_n] \le \frac{b-1}{2} n^2$$

**证明**. 令$Y_n = X_n + \frac{1}{b-1}$，则
$$
\begin{aligned}
E[Y_n^2] &= \sum_c Pr[C_n = c]\left(\frac{b^c}{b-1}\right)^2 \\\\
&= \sum_c Pr[C_{n-1} = c] \left( b^{-c}\left(\frac{b^{c+1}}{b-1}\right)^2 + (1-b^{-c})\left(\frac{b^c}{b-1}\right)^2 \right) \\\\
&= \sum_c Pr[C_{n-1} = c] \left( \left(\frac{b^c}{b-1}\right)^2 + \frac{b^{-c}}{(b-1)^2} \left(\left(b^{c+1}\right)^2 - \left(b^c\right)^2 \right) \right) \\\\
&= E[Y_{n-1}^2] + \sum_c Pr[C_{n-1} = c] \frac{b^{-c}}{(b-1)^2}(b^{c+1}-b^c)(b^{c+1}+b^c) \\\\
&= E[Y_{n-1}^2] + (b+1) \sum_c Pr[C_{n-1} = c] \frac{b^c}{b-1} \\\\
&= E[Y_{n-1}^2] + (b+1) E[Y_{n-1}] \\\\
&= E[Y_{n-1}^2] + (b+1)(n-1) \\\\
&= E[Y_0^2] + (b+1)\sum_{i=0}^{n-1} i
\end{aligned}
$$
由于$Y_0 = \frac{1}{b-1}$，因此
$$E[X_n^2]=E\left[\left(Y_n - \frac{1}{b-1}\right)^2 \right] \le E[Y_n^2 - Y_0^2] = \frac{b+1}{2}n(n-1)$$

从而
$$var[X_n] \le E[X_n^2] - E[X_n]^2 = \frac{b-1}{2}n^2$$

**定理**. MorrisCounter对真实频数$n$的近似估计$X_n$满足

$$Pr[|X_n - n| \le \epsilon n] \ge 1 - \delta$$

此时所需的空间复杂度为$\mathrm{O}\left(\log \frac{1}{\epsilon} + \log \frac{1}{\delta} + \log\log\epsilon^2\delta n \right)$。

**证明**. 根据*引理1*和*引理2*，由Chebyshev不等式可得
$$Pr[|X_n - n| \ge \epsilon n] \le \frac{b-1}{2}n^2/\epsilon^2n^2 = \frac{b-1}{2}\epsilon^2$$

令$b \le 1 + 2\epsilon^2\delta$，那么我们就可以以至少$1-\delta$的概率提供对频数相对误差为$\epsilon$的估计。

此时MorisCounter算法所需的空间复杂度为$\mathrm{O}\left(\log \frac{1}{\epsilon} + \log \frac{1}{\delta} + \log\log\epsilon^2\delta n \right)$。



## 参考文献

<div id="ref-1"></div> 

[1] R. Morris. *Counting large numbers of events in small registers*. In Communications of the ACM, vol 21, issue 10, pp. 840-842, 1978.
