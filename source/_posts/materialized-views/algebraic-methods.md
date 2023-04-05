---
title: "物化视图维护的代数方法"
date: 2023-04-05
categories:
  - "物化视图"
tags:
  - "ETL"
  - "物化视图"
toc: true
---

为了保证与数据源的一致性，当数据源发生变化时，物化视图需要进行及时的更新。很多时候，数据源的变化相比于整体而言是较小的。此时使用增量更新的方式来对物化视图进行维护会更加高效。本文主要介绍维护物化视图的代数方法。这些方法可能在某些算子上无法提供最高效的维护方法，但在语义上比较直观。



## 研究历史

对物化视图维护的研究最早可追溯到1980年代。这些早期的工作都局限在特定的数据库模型上。[1] 对函数式关联数据模型 (functional association data model) 上的视图维护方法进行了研究。而 [2] 只支持无环数据库 (acyclic database) 上的视图。这些视图先将数据表中所有关系表进行自然连接，之后进行映射；无法支持选择操作，也无法对部分表进行连接。

[3] 是第一个对一般物化视图的维护进行研究的工作，他们的方法可以对包含选择、映射和连接等操作在内的视图进行维护。[4] 则进一步提出使用代数方法 (algebraic method) 来描述物化视图维护的方法。在代数方法中，输入表$R$的更新被描述为一对增量表：$\nabla R$和$\Delta R$。其中$\nabla R$表示$R$中被删除的记录，而$\Delta R$表示$R$中新插入的记录。我们使用一组等式来描述这些变化是如何在算子之间传播的。在 [4] 中，作者提供了*集合代数 (set algebraic)* 上选择、映射、笛卡尔积、并、差、交和内连接等算子的变化传播方程 (change propagation equation)。[5] 则给出了*包代数 (bag algebraic)* 上相关算子的变化传播方程。

[6] 为包含外连接的视图提供了维护方法，但他们的方法并不是基于代数形式的。[7] 给出了描述维护半连接和外连接视图的代数方法。[8] 给出了基于change table方法对外连接视图进行更新的方法，但他们的方法显然是错误的。

相较于其他算子，聚合算子的变化传播很难通过代数形式进行描述。首先，聚合算子本身的代数描述就比困难。其次，部分聚合操作（如 median）并不满足分配律 (distributive property)，因此无法很难描述输出变化和输入变化之间的关系。 [5] 给出了聚合算子的变化传播方程，但他们的聚合算子无法支持分组语句 (group-by)，并且只能作为视图中最后一个算子。[8] 通过通用映射 (general projection) 来对聚合操作进行描述，给出了一般情况下可分配的聚合算子的变化传播方程。除此之外，其他在维护聚合视图上的工作 [10, 11]，都只描述了具体的过程，而没有给出形式化描述。

在维护视图的代数方法中，如果我们向后传播的插入和删除中存在重叠的记录，那么显然会导致后面的增量更新存在不必要的冗余计算。我们希望向下游传播的变化是最小化的，即$\Delta R \cap \nabla R = \emptyset$。[4] 虽然给出了常见算子的变化传播方法并号称这些方法满足最小化条件，但实际并非如此。[12] 指出了 [4] 中的错误，并给出了一个通用的构建最小化结果变化的方法。

虽然代数方法可以用于解决一般情况下的视图更新问题，但在实际中我们的视图通常会多个嵌套的算子。此时使用代数方法去逐个算子逐个输入的去计算视图的变化效率会非常低。此时，我们常常需要一些更加高效但没有那么纯粹 (pure) 的方法来计算视图的更新 [9]。



## 视图更新的代数方法



### 选择 (Select)

选择算子的变化传播方程如下

$$ \sigma_p(R \cup \Delta R) = \sigma_p(R) + \sigma_p(\Delta R)$$

$$\sigma_p(R - \nabla R) = \sigma_p(R) - \sigma_p(\nabla R)$$



### 映射 (Project)

映射算子的变化传播方程如下

$$\pi_A(R \cup \Delta R) = \pi_A(R) \cup \pi_A(\Delta R)$$

$$\pi_A(R - \nabla R) = \pi_A(R) - \pi_A(\nabla R)$$



### 并 (Union)

集合代数和包代数下并操作有着不同的语义。我们使用`UNION`和`UNION ALL`来区分这两种不同的操作。

#### UNION ALL

我们使用$R \cup S$来表示表$R$和$S$在包语义下的并操作。如果一个记录$t$在$R$中出现了$n$次，在$S$中出现了$m$次。如果$n > 0$或者$m>0$，那么$t \ in R \cup S$，并且在$R\cup S$中出现$n + m$次。

`UNION ALL`的变化传播方程如下

$$(R - \nabla R) \cup S = (R \cup S) - \nabla R$$

$$(R \cup \Delta R) \cup S = (R \cup S) \cup \Delta R$$

#### UNION 

我们使用$R\sqcup S$表示表$R$和$S$在集合语义下的并操作。如果一个记录$t$在$R$出现了或者在$S$中出现了，那么$t$也会出现在$R \sqcup S$中，并且只出现一次。



当$R$中某个记录$r$被删除时，对于$R \sqcup S$可能存在以下两种情况

* 如果$r$在$R - \nabla R$或者$S$中出现了，那么$r$仍然会在最终的结果中出现。$r$的删除并不会对最终的结果产生任何影响。
* 如果$r$既没有出现在$R - \nabla R$中，也没有出现在$S$中，那么从$R$中删除$r$之后，$r$也会从最终的结果中被删除。我们使用$\nabla R \ominus ((R - \nabla R) \sqcup S)$ 来表示$\nabla R$中没有出现在$R - \nabla R$和$S$中的记录。

根据上面的讨论可知：

$$ (R - \nabla R) \sqcup S = (R \sqcup S) - (\nabla R \ominus ((R - \nabla R) \sqcup S))$$



当$R$中添加某个记录$r$时，对于$R \sqcup S$可能存在以下两种情况

* 如果$r$已经在$R$或者$S$中出现了，那么$r$已经在最终的结果中出现了。插入$r$并不会对最终的结果产生任何影响。
* 如果$r$既没有出现在$R$中，也没有出现在$S$中，那么$r$就需要添加到最终的结果中。我们使用$\Delta R \ominus (R \sqcup S)$来表示$\Delta R$中既没有出现在$R$也没有出现在$S$中的记录。

根据上面的讨论可知

$$(R \cup \Delta R) \sqcup S = (R \sqcup S) \cup (\Delta R \ominus (R \sqcup S))$$



### 交 (Intersect)

和并操作一样，交操作需要区分集合语义和包语义。我们使用`INTERSECT ALL`表示包语义下的并操作，`INTERSECT`表示集合语义下的并操作。

#### INTERSECT ALL

我们使用$R \cap S$表示表$R$和$S$在包语义下的交操作。如果一个记录$t$在$R$中出现了$n$次，在$S$中出现了$m$次，并且$n > 0, m > 0$，那么$t \in R \cap S$，并在$R \cap S$中出现$\min(n, m)$次。



当$R$中删除一个记录$r$时

- 如果$n > m$，那么在删除掉$r$不会对最终的结果产生影响。
- 如果$n \le m$，那么在删除掉$r$之后，应该在最终的结果中减少一次$r$的出现。

根据上面的讨论可知

$$ (R - \nabla R) \cap S = (R \cap S) - (\nabla R - (R - S)) $$



当$R$中插入一个记录$r$时

* 如果$n \ge m$，那么在添加$r$时不会对最终的结果产生影响。
* 如果$n < m$，那么在添加$r$后，应该在最终的结果中增加一次$r$的出现。

根据上面的讨论可知

$$ (R \cup \Delta R) \cap S = (R \cap S) \cup (\Delta R \cap (S - R)) $$

#### INTERSECT

我们使用$R \sqcap S$表示表$R$和$S$在集合语义下的交操作。如果一个记录$t$在$R$中出现了，在$S$中也出现了，那么$t \in R \sqcap S$，并在$R \sqcap S$中出现一次。



参考`UNION`的讨论方式，可得`INTERSECT`的变化传播方程为

$$(R - \nabla R) \sqcap S = (R \sqcap S) - (\nabla R \sqcap (S \ominus (R - \nabla R)))$$

$$(R \cup \Delta R) \sqcap S = (R \sqcap S) \cup (\Delta R \sqcap (S \ominus R)))$$



### 差 (Minus)

和并操作一样，差操作需要区分集合语义和包语义。我们使用`MINUS ALL`表示包语义下的差操作，`MINUS`表示集合语义下的差操作。

#### MINUS ALL

`MINUS ALL`按照包语义执行差操作。我们使用$R - S$表示表$R$和$S$在包语义下的差操作。如果一个记录$t$在$R$中出现了$n$次，在$S$中出现了$m$次，并且$n > m$，那么$t \in R - S$，并且在$R - S$中出现$n - m$次。

`MINUS ALL`的变化传播方程如下

$$(R - \nabla R) - S = (R - S) - \nabla R$$

$$(R \cup \Delta R) - S = (R - S) \cup \Delta R$$

#### MINUS

`MINUS`按照集合语义执行差操作。我们使用$R \ominus S$表示表$R$和$S$在集合语义下的差操作。如果一个记录$t$在$R$中出现了，但在$S$中没有出现，那么$t \in R \ominus S$，并且在$R \ominus S$只出现一次。



当我们从$R$中删除一个记录$r$时

* 如果$r$没有在$S$中出现，那么$r$应该会出现在原来的结果中。
  * 如果$r$也没有在$R - \nabla R$中出现，那么将$r$从$R$中删除之后，$r$就需要从最终的结果中删除掉。
  * 如果$r$出现在$R - \nabla R$中，那么将$r$从$R$中删除之后，并不会对最终的结果产生任何影响。
* 如果$r$出现在$S$中，那么$r$本来就不存在于原来的结果中。删除$r$并不会对最终的结果产生任何影响。

根据上面的讨论可知，当$r$既没有出现在$S$中也没有出现在$R - \nabla R$中时，删除$r$会导致$r$从最终的结果中删除。因此

$$(R - \nabla R) \ominus S = (R \ominus S) - (\nabla R \ominus ((R - \nabla R) \sqcup S))$$



当我们向$R$中插入一个新记录$r$时

* 如果$r$已经出现在$S$中了，那么$r$不会出现在原来的结果中，插入$r$也不会导致最终结果的变化。
* 如果$r$没有在$S$中出现
  * 如果$r$已经出现在$R$中了，那么插入$r$也不会导致最终结果的变化。
  * 如果$r$没有出现在$R$中，那么插入$r$后$r$会出现在最终结果中。

根据上面的讨论可知，当$r$既没有出现在$S$也没有出现在$R$中时，插入$r$会导致$r$添加到最终的结果中。因此

$$(R \cup \Delta R) \ominus S = (R \ominus S) \cup (\Delta R \ominus (R \sqcup S))$$



类似的，我们对$S$插入或删除的情况进行讨论，可得

$$ R \ominus (S - \nabla S) = (R \ominus S) \cup (\nabla S \sqcap (R \ominus (S - \nabla S)))$$

$$R \ominus (S \cup \Delta S) = (R \ominus S) - (\Delta S \sqcap (R \ominus S))$$



### 内连接 (Inner Join)

我们用$R \Join_p S$表示表$R$和表$S$按照连接条件$p$进行的连接操作。

内连接的变化传播方程如下

$$(R - \nabla R) \Join_p S = (R \Join_p S) - (\nabla R \Join_p S)$$

$$(R \cup \Delta R) \Join_p S = (R \Join_p S) \cup (\Delta R \Join_p S)$$



### 左外连接 (Left Outer Join)

我们使用$R ⟕_p S$表示表$R$和$S$按照条件$p$进行的左外连接操作。左外连接$R ⟕_p S$除了返回$R \Join S$之外，还会将$R$中无法和$S$进行连接的记录用null补齐$S$中的字段返回。即

$$R ⟕_p S = (R \Join_p S) \cup ((R \bar{\ltimes} S) \times \{d_S\})$$

其中$d_S$ 是一个和$S$的字段一致并且所有列都为null的记录。



当$R$中某个记录$r$被删除时

* 如果在表$S$中存在$s$使得$p(r,s)$成立，那么$(r, s)$出现在原来的结果中，此时我们需要将$(r,s)$从最终的结果中删除。
* 如果在表$S$中不存在$s$使得$p(r,s)$成立，那么$(r, d_S)$ 出现在原来的结果中。此时我们需要将$(r, d_S)$从最终的结果中删除。

根据上面的讨论可知

$$(R - \nabla R) ⟕_p S = (R ⟕_p S) - (\nabla R ⟕_p S)$$



当$R$中添加某个记录$r$时

* 如果在表$S$中存在$s$使得$p(r,s)$成立，那么$(r,s)$会出现在最终的结果中。
* 如果在表$S$中不存在$s$使得$p(r,s)$成立，那么$(r, d_S)$会出现在最终的结果中。

根据上面的讨论可知

$$(R \cup \Delta R) ⟕_p S = (R ⟕_p S) \cup (\Delta R ⟕_p S)$$



类似的，我们对$S$插入或删除的情况进行讨论，可得

$$R ⟕_p (S - \nabla S) = ((R ⟕_p S) - (R \Join_p \nabla S)) \cup (((R \ltimes_p \nabla S) \bar{\ltimes}_p (S - \nabla S)) \times \{d_S\})$$

$$R ⟕_p (S \cup \Delta S) = ((R ⟕_p S) \cup (R \Join_p \Delta S)) - (((R \ltimes_p \Delta S) \bar{\ltimes}_p S) \times \{d_S\})$$



### 全外连接 (Full Outer Join)

我们使用$R ⟗_p S$表示表$R$和$S$按照条件$p$进行的全外连接操作。全外连接$R ⟗_p S$除了返回$R \Join S$之外，还会将$R$中无法和$S$进行连接的记录用null补齐$S$中的字段返回，将$S$中无法和$R$进行连接的记录用null补齐$R$中的字段后返回。即

$$R ⟗_p S = (R \Join_p S) \cup ((R \bar{\ltimes} S) \times \{d_S\}) \cup ((S \bar{\ltimes} R) \times \{d_R\})$$

其中$d_S$ 是一个和$S$的字段一致并且所有列都为null的记录，$d_R$是一个和$R$的字段一致并且所有列都为null的记录。



和左外连接的情况类似，我们可得

$$(R - \nabla R) ⟗_p S = ((R ⟗_p S) - (\nabla R ⟕_p S)) \cup (((S \ltimes_p \nabla R) \bar{\ltimes}_p (R - \nabla R)) \times \{d_R\})$$

$$(R \cup \Delta R) ⟗_p S = ((R ⟗_p S) \cup (\Delta R ⟕_p S)) - (((S \ltimes_p \Delta R) \bar{\ltimes}_p R) \times \{d_R\})$$



### 半连接 (Semi Join)

我们使用$R \ltimes_p S$表示表$R$和$S$按照条件$p$​进行的半连接操作。半连接$R \ltimes_p S$返回$R$中和$S$可以按照连接条件$p$进行连接的记录，即

$$ R \ltimes_p S = \{ r | r \in R, \exists s \in S, p(r, s) \}$$ 



当$R$中某个记录$r$被删除时

1. 如果在表$S$中存在$s$使得$p(r, s)$成立，那么$r$出现在原来的结果中。此时我们需要将$r$从最终的结果中删除。
2. 如果在表$S$中不存在$s$使得$p(r, s)$成立，那么$r$没有出现在原来的结果中，因此也就不会对最终的结果产生影响。

根据上面的讨论可知

$$(R - \nabla R) \ltimes_p S = (R \ltimes_p S) - (\nabla R \ltimes_p S)$$



当$R$中添加某个记录$r$时

1. 如果在表$S$中存在$s$使得$p(r, s)$成立，那么$r$会出现在最终的结果中。
2. 如果在表$S$中不存在$s$使得$p(r, s)$成立，那么$r$不会出现在最终的结果中。

根据上面的讨论可知

$$(R \cup \Delta R) \ltimes_p S = (R \ltimes_p S) \cup (\Delta R \ltimes_p S)$$



类似的，我们对$S$插入或删除的情况进行讨论，可得

$$R \ltimes_p (S - \nabla S) = (R \ltimes_p S) - ((R \ltimes_p \nabla S) \bar{\ltimes}_p (S - \nabla S))$$

$$ R \ltimes_p (S \cup \Delta S) = (R \ltimes_p S) \cup ((R \ltimes_p \Delta S) \bar{\ltimes}_p S))$$



### 反半连接 (Anti Semi Join)

我们使用$R \bar{\ltimes}_p S$表示表$R$和$S$按照条件$p$进行的反半连接操作。反半连接$R \bar{\ltimes}_p S$返回$R$中可以和$S$按照连接条件$p$进行连接的记录，即

$$ R \bar{\ltimes}_p S = \{ r | r \in R, \nexists s \in S, p(r, s) \}$$ 



和半连接的情况类似，我们可得

$$(R - \nabla R) \bar{\ltimes}_p S = (R \bar{\ltimes}_p S) - (\nabla R \bar{\ltimes}_p S)$$

$$(R \cup \Delta R) \bar{\ltimes}_p S = (R \bar{\ltimes}_p S) \cup (\Delta R \bar{\ltimes}_p S)$$

$$R \bar{\ltimes}_p (S - \nabla S) = (R \bar{\ltimes}_p S) \cup ((R \ltimes_p \nabla S) \bar{\ltimes}_p (S - \nabla S))$$

$$R \bar{\ltimes}_p (S \cup \Delta S) = (R \bar{\ltimes}_p S) - (R \ltimes_p \Delta S)$$



## Related Work

[1] S. Koenig et al. *A transformational framework for the automatic control of derived data*. In VLDB 1981, pp. 306-318.

[2] O. Shmueli et al. *Maintenance of views*. In SIGMOD 1984, pp. 240-255.

[3] J. Blakeley et al. *Efficiently updating materialied views*. In SIGMOD 1986, pp. 61-71. 

[4] X. Qian et al. *Incremental recomputation of active relational expressions*. In TKDE 1991 vol. 3(3) pp. 337-341.

[5] T. Griffin et al. *Incremental maintenance of views with duplicates*. In SIGMOD 1995, pp. 328-339.

[6] A. Gupta et al. *Maintenance and self-maintenance of outerjoin views*. In Next Generation Information Technology and Systems 1997.

[7] T. Griffin et al. *Algebraic change propagation for semijoin and outerjoin queries*. In SIGMOD Record 1998 vol. 27(3), pp. 22-27.

[8] H. Gupta et al. *Incremental maintenance of aggregate and outerjoin expressions*. In Information Systems 2006, vol. 31(6), pp. 435-464.

[9] P. Larson et al. *Efficient maintenance of materialized outer-join views*. In ICDE 2007, pp. 56-65.

[10] I. Mumick et al. *Maintenance of data cubes and summary tables in a warehouse*. In SIGMOD Record 1997, vol. 26(2), pp.100-111.

[11] T. Palpanas et al. *Incremental maintenance for non-distributive aggregate functions*. In VLDB 2002, pp. 802-813.

[12] T. Griffin et al. *An improved algorithm for the incremental recomputation of active relational expressions*. In TKDE 1997, vol. 9(3) pp. 508-511.



