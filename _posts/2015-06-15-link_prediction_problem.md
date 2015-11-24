---
layout: post
title:  "Link Prediction Problem"
date:   2015-06-15 20:40:14
tags: algorithm
mathjax: true
---


## 1

最近在看一些好友推荐的东西，所以就多google了一下。

看到一篇论文[《The Link Prediction Problem for Social Networks》](http://www.cs.cornell.edu/home/kleinber/link-pred.pdf)，感觉写的很不错。

（对这种形式化的描述没抵抗力 :-D）

所以就是做个笔记吧。

## 2

首先*social networks*可以理解为图，其中*nodes*表示人或者其他实体，*edges*标示交互、协作或者关系。

那么好友推荐的问题就可以理解为基于原有的好友关系图结构，预测可能出现的新的关系。

更一般的，可以理解为*link prediction problem*，基于原有的图结构，预测图的演化。

>the *link prediction problem*：
>
>Given a snapshot of a social network at time *t*, we seek to accurately predict the edges that will be added to the network during the interval from time $t$ to a given future time $t'$.

当然现实中的交互和关系有很多原因，单单从图结构中是分析不出来的。

但是

>We find that a number of proximity measures lead to predictions that outperform chance by factors of 40 to 50, indicating that the network topology does indeed contain latent information from which to infer future interactions.

（是这样么？ 暂且相信一下吧，不然就没法继续读下去了 :-D）

## 3

形式化这个问题。

social network: $G = \langle V, E \rangle$

each edge: $e = \langle u, v \rangle \in E$

表示$u,v$在特定时间$t(e)$发生交互，其中多次交互记录为并行的边。

$G[t, t']$表示某时间段内$(t < t')$的子图。

另外这篇文章为了评估不同的算法还是定义训练和测试的方法：

>We choose four times $t_0 < t'_0 < t_1 < t'_1$, and give an algorithm access to the network $G[t_0,t'_0]$; it must then output a list of edges, not present in $G[t_0,t'_0]$, that are predicted to appear in the network $G[t_1,t'_1]$. We refer to $[t_0,t'_0]$ as the *training interval* and $[t_1,t'_1]$ as the *test interval*.
>
>Thus, in evaluating link prediction methods, we will generally use two parameters $\mit{κ_{training}}$ and $\mit{κ_{test}}$ (each set to 3 below), and define the set **Core** to be all nodes incident to at least $\mit{κ_{training}}$ edges in $G[t_0, t'_0]$ and at least $\mit{κ_{test}}$ edges in $G[t_1,t'_1]$. We will then evaluate how accurately the new edges between elements of **Core** can be predicted.
>
>We define the training interval to be the three years $[1994, 1996]$, and the test interval to be $[1997, 1999]$. We denote the subgraph $G[1994, 1996]$ on the training interval by $G_{collab} := \langle A, E_{old} \rangle $, and use $E_{new}$ to denote the set of edges $\langle u, v \rangle$ such that $u, v \in A$, and $u, v$ co-author a paper during the test interval but not the training interval -- these are the new interactions we are seeking to predict.

评估的方法如下：

>Each link predictor $p$ that we consider outputs a ranked list $L_p$ of pairs in $A×A−E_{old}$; these are predicted new collaborations, in decreasing order of confidence. For our evaluation, we focus on the set **Core**, so we define $E^∗_{new} := E_{new} \cap (Core \times Core)$ and $n := |E^∗_{new}|$. Our performance measure for predictor $p$ is then determined as follows: from the ranked list $L_p$, we take the first $n$ pairs in $Core \times Core$, and determine the size of the intersection of this set of pairs with the set $E^∗_{new}$.
>

## 4

定义方法的输入输出。

输入：$G_{collab}$

输出：$score(x, y)$降序的列表

基本上可以理解为通过原有的图结构计算节点$x,y$之间的proximity。

最基本的方法就是计算$G_{collab}$中$x, y$的最短路径，为了保持结果降序排列，对路径长度取负值，其中等于1的路径属于训练图。

## 5

Methods based on node neighborhoods

$\Gamma(x)$表示图$G_{collab}$中节点$x$的邻居的集合。

直觉上来说，相同的邻居更多的连个节点更容易产生关系，这类方法就是基于这个假设。

### Common neighbors

$score(x, y) := \mid\Gamma(x)\cap\Gamma(y)\mid $

### Jaccard's coefficient and Adamic/Adar

Jaccard's coefficient通常用于信息检索中的相似度测量，

$score(x, y) := \mid\Gamma(x)\cap\Gamma(y)\mid/\mid\Gamma(x)\cup\Gamma(y)\mid$

Adamic/Adar加大了稀有特征的权重，

$score(x,y) := \sum\nolimits_{z\in\Gamma(x)\cap\Gamma(y)}\frac{1}{\log\mid\Gamma(z)\mid}$

### Preferential attachment

该方法的基本的假设是节点涉及新边的概率与其邻居的数量成正比，

$score(x,y) := \mid\Gamma(x)\mid\cdot\mid\Gamma(y)\mid$

## 6

Methods based on the ensemble of all paths

这类方法基于最短路径的思路，不过是考虑了两点之间所有路径。

### Katz

$score(x,y) := \sum\limits_{l=1}^{\infty}\beta^l\cdot\mid paths_{x,y}^{\langle l \rangle} \mid$

$paths_{x,y}^{\langle l \rangle} := \{paths\ of\ length\ exactly\ l\ from\ x\ to\ y\}$

当$\beta$值比较小的时候，结果基本与common neighbors方法类似。

此外该方法还分为加权、不加权两种：

weighted: $paths_{x,y}^{\langle 1 \rangle} := number\ of\ collaborations\ between\ x,y.$

unweighted: $paths_{x,y}^{\langle 1 \rangle} := 1\ iff\ x\ and\ y\ collaborate.$

### Hitting times, PageRank, and variants

该方法的思想是在$G_{collab}$上进行*random walk*

*hitting time* $H_{x,y}$标示从$x$到$y$的期望的步数

*commute time* $C_{x,y} := H_{x,y} + H_{y,x}$

但是当$y$的*stationary probability* $\pi_y$比较大的时候，对于任意$x$，$H_{x,y}$的值都很小，所以做一点儿归一化。

$score(x,y) := -H_{x,y}\cdot\pi_y$

$score(x,y) := -(H_{x,y}\cdot\pi_y + H_{y,x}\cdot\pi_x)$

但是这种测量方法还有一个问题，其测量值对图部分的结构更敏感，所以在*random walk*中加入固定的概率$\alpha$退回。

>rooted PageRank
>
>stationary distribution weight of $y$ under the following random walk:
>
>with probability $\alpha$, jump to $x$.
>
>with probability $1-\alpha$, go to random neighbor of current node.
>

### SimRank

$score(x,y) := \begin{equation}\left \{\begin{array}\\\gamma\cdot\frac{\sum\nolimits_{a\in\Gamma(x)}\sum\nolimits_{b\in\Gamma(y)}score(a,b)}{\mid\Gamma(x)\mid\cdot\mid\Gamma(y)\mid} & otherwise \\1 & if\ x = y\end{array}\right.\end{equation}$

$\gamma\in[0,1]$

>SimRank can also be interpreted in terms of a random walk on the collaboration graph: it is the expected value of $\gamma^l$, where $l$ is a random variable giving the time at which random walks started from $x$ and $y$ first meet.

##7

Higher-level approaches

三个meta-approaches，可以结合任一上述方法。

### Low-rank approximation

上述算法都可以转换为矩阵运算，那么就可以用SVD降秩，这也可以看做为一种noise-reduction的方法。

### Unseen bigrams

>Link prediction is akin to the problem of estimating frequencies for unseen bigrams in language modeling—pairs of words that co-occur in a test corpus, but not in the corresponding training corpus.
>
>we can augment our estimates for $score(x, y)$ using values of $score(z, y)$ for nodes $z$ that are “similar” to $x$.

$score_{unweighted}^*(x,y) := \mid\{z:z\in\Gamma(y)\cap S_x^{\langle\delta\rangle}\}\mid$

$score_{weighted}^*(x,y) := \sum\nolimits_{z\in\Gamma(y)\cap S_x^{\langle\delta\rangle}}score(x,z)$

$S_x^{\langle\delta\rangle}$表示基于$score(x,\cdot)$与$x$最相关的$\delta$个节点。

### Clustering

通过聚类删除一些tenuous的边，然后在cleaned-up的子图上进行计算。

##8

一些结论

>the small world problem is really a problem. The shortest path between two scientists in wholly unrelated disciplines is often very short (and very tenuous).
>
>Overall, the basic graph distance predictor is not competitive with most of the other approaches studied; our most successful link predictors can be viewed as using measures of proximity that are robust to the few edges that result from rare collaborations between fields.

AA还是不错的。

>The small world problem suggests that there are many pairs with graph distance two that will not collaborate, but we also observe the dual problem: many pairs that collaborate are at distance larger than two.

