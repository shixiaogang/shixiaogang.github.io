---

title: "Misra-Gries频数估计算法"
date: 2023-04-29
categories:
  - "概要算法"
  - "频数估计"
tags:
  - "数据流"
  - "概要算法"
  - "频数估计"
toc: true
---



Misra-Gries算法最早由J. Misra和D. Gries在1982年提出 [1]。Misra-Gries算法可以看成是对Majority算法的一个扩展，可以对数据流中的频数提供相对误差为$\epsilon$的估计，使用的空间复杂度为$\mathrm{O}(1/\epsilon)$。

到了2000年左右，随着对数据流研究的再次兴起，[2] 和 [3] 又重新提出了类似的算法来用于频数估计。和原始论文不同，[2] 使用了哈希表而非平衡搜索树来保证频繁项。[3] 则通过多个链表来提高清理记录时的效率，使其在最坏情况下仍然可以保证$O(1)$的开销。

原始论文侧重于频繁项的估计，并没有对频数估计的误差进行分析。[4] 证明了当$k = 1/\epsilon$时，频数估计的相对误差为$\epsilon$。[5] 则表明Misra-Gries算法的估计误差仅取决于那些长尾的数据项。考虑到真实环境中的数据分布通常是倾斜的（如zipf分布），这个结论可以进一步提高收紧了估计误差的边界。

[5] 进一步扩展了Misra-Gries算法，使其可以支持记录有不同权重的场景。[6] 则优化了权重更新场景下更新和合并的效率。[5] 还提供了Misra-Gries算法的合并操作，而 [7] 则对合并的空间复杂度提供了更强的边界。[7] 同时也证明了Misra-Gries算法本质上是和SpaceSaving算法等价的。



## 算法实现

我们在Misra-Gries算法中维护了一组记录$(e, f)$，其中$e$是数据流中的数据项，$f$是对$e$的频数估计。我们将这组记录记为$\mathcal{D}$。



Misra-Gries算法的更新算法如下所示。当一个新元素$i$到达时，我们首先检查其是否已经在$\mathcal{D}$中存在。如果已经存在的话，那么我们直接将其对应的频数$f_i$加1；否则，我们在$\mathcal{D}$中添加一个新记录$(e, 1)$。如果$\mathcal{D}$中记录的数目超过了设定的阈值$k$，那么我们就需要对$\mathcal{D}$进行清理。我们去除当前$\mathcal{D}$中所有频数的最小值$f_m$，并将所有频数都减去$f_m$。如果一个数据项的频数变为$0$，那么我们就将其从$\mathcal{D}$中删除。



{% listing title:Misra-Gries算法更新操作 %}

if $i \in \mathcal{D}$, then

   $f_i \gets f_i + 1$

else 

​    $\mathcal{D} \gets \mathcal{D} \cup \\{i\\}$

​    $f_m = \min \\{ f_e | e \in \mathcal{D} \\}$

​    for all $j \in \mathcal{D}$

​        $f_j \gets f_j - f_m$

​        if $f_j = 0$, then

​            $\mathcal{D} \gets \mathcal{D} - \\{j \\}$

​        end if

​    end for

end if

{% endlisting %}



Misra-Gries算法的查询操作如下所示。当我们需要查询某个数据项的频数时，如果这个数据项存在于$\mathcal{D}$中，我们就返回其在$\mathcal{D}$中对应的频数；否则，我们直接返回$0$。

{% listing title:Misra-Gries算法查询操作 %}

if $e \in \mathcal{D}$

​    return $f_e$

else

​    return $0$

end if

{% endlisting %}



Misra-Gries算法也支持合并操作。在合并时，我们任意选择一个频繁集来作为基础，然后再将另一个频繁集中的记录逐个添加进去。Misra-Gries算法的合并操作如下所示：

{% listing title:Misra-Gries算法合并操作 %}

$\mathcal{D} \gets \mathcal{D}_a$



for all $i \in \mathcal{D}_b$

​    if $i \in \mathcal{D}$, then

​        $f_i \gets f_i + f_{i, b}$

​    else

​        $\mathcal{D} \gets \mathcal{D} \cup \\{i \\}$

​        $f_i \gets f_{i, b}$

​    end if

​    if $|\mathcal{D}| > k$, then

​        $f_m \gets \min \\{ f_e | e \in \mathcal{D} \\}$

​        for all $j \in \mathcal{D}$

​            $f_j \gets f_j - f_m$

​            if $f_j = 0$, then

​                $\mathcal{D} \gets \mathcal{D} - \\{j\\}$

​            end if

​        end for

​    end if

end for

{% endlisting %}



## 算法分析

**定理**. Misra-Gries算法对真实频数$f$提供的估计$\hat{f}$满足

$$ 0 \le f - \hat{f} \le \epsilon n $$

其中$n$为所有数据项的频数之和。此时所需的空间复杂度为$\mathrm{O}(1/\epsilon)$。

**证明**. 令$n$为数据流中已经出现的记录数目，$n'$为保存在$\mathcal{D}$中所有频数的和。

我们对于频数估计的误差来自于我们对$\mathcal{D}$的清理操作。每次清理时，我们会将$\mathcal{D}$中的记录都减去一个相同的值。因此在整个计算过程中，受到影响的数据项数目一定大于等于$k$。因此，我们有

$$ 0 \le f - \hat{f} \le \frac{n - n'}{k} \le \frac{n}{k} $$

令$k = 1/\epsilon$，则

$$0 \le f - \hat{f} \le \epsilon n$$



## 参考文献

[1] J. Misra, D. Gries. Finding repeated elements. In Science of Computer Programming 1982, vol. 2(2), pp. 143-152.

[2] E. Demaine et al. Frequency estimation of internet packet streams with limited space. In ESA 2002, pp. 348-360.

[3] R. Karp et al. A simple algorithm for finding frequent elements in streams and bags. In TODS 2003, vol. 28(1), pp. 51-55.

[4] P. Bose et al. Bounds for frequency estimation of packet streams. In SIROCCO 2003.

[5] R. Berinde et al. Space-optimal heavy hitters with strong error bounds. In PODS 2009, pp. 157-166.

[6] D. Anderson et al. A high-performance algorithm for identifying frequent items in data streams. In IMC 2017, pp. 268-282.

[7] P. Agarwal et al. Mergeable summaries. In PODS 2012, pp. 23-34. 
