---
layout:     post
title:      LambdaRank 学习及 lgb 中使用
subtitle:
date:       2022-07-19 00:00:00
author:     mzl
catalog:    true
tags:
    - lambdarank
    - lightgbm
    - 学习
---

{:toc}

# 1.背景

前段时间打 kaggle 比赛 Foursqaure - Location Match 时, 使用 lgb 的 lambdarank 减少数据量, 实现了减少内存 + 加快计算的目的, 但对 lambdarank 并没有深入学习. 今天打算阅读一下论文并记录一下使用体会.

## 训练代码及关键点

### 关键点及训练代码

排序模型有一个关键点, 即构建 label 时要表明哪些 retrieval 是同一个 query 的, 所以要有一个标识, 在 lgb 中要送到 group/eval_group 这两个参数中

```python
rank_params = {
    "boosting_type": "gbdt",
    "objective": "lambdarank",
    "metric": "ndcg",
    "max_depth": 8,
    "num_leaves": 40,
    "learning_rate": 0.02,
    "colsample_bytree": 1,
    "min_child_samples": 18,
    # "subsample": 0.9,
    # "subsample_freq": 2,
    "reg_alpha": 0.5,
    "reg_lambda": 0.5,
    "cat_smooth": 0,
    "n_estimators": 3000,
    # "importance_type": "gain"
}

import lightgbm as lgb

rank_model = lgb.LGBMRanker(**rank_params)

import pickle

rank_model.fit(
    X=train_rank_df[features],
    y=train_rank_df[label],
    group=train_rank_df.groupby("id_1")["id_1"].count().to_numpy(),
    eval_set=[(test_rank_df[features], test_rank_df[label])],
    eval_group=[test_rank_df.groupby("id_1")["id_1"].count().to_numpy()],
    eval_at=[10],
    early_stopping_rounds=50,
    verbose=50,
)
```

### 未留意到的部分

lgb 中有一些参数未留意到, 具体如何使用及调优未明确: lambdarank_truncation_level, lambdarank_norm, label_gain

1. lambdarank_truncation_level: 默认为 20, 计算 DCG 时截断计算, 指 "truncation level", 可参考 LambdaMART paper Sec 3.
2. lambdarank_norm: 默认为 true, 设为 true 的话, 会 normalize 不同 query 的 lambda 值, 从而提升不均衡数据集的性能
3. label_gain: 默认$0,1,3,7,15,31,63,\dots,2^{30}-1$, 不同 labels 的相关增益, 该参数的调参是悬学.

# 2.论文阅读

## 2.1 摘要

### 2.1.1 要解决的问题

IR(Information Retrieval)领域的质量指标很难直接优化, 原因是这些指标依赖模型对指定 query 的 retrieval 的评分. 因此, cost 对模型参数的导数要么定义为0, 要么未定义.

### 2.1.2 创新点

本文提出 LambdaRank, 通过隐式地使用 cost function 来避免这些困难. 同时我们给出这个隐式成本函数是凸的充要条件, 并证明该方法有简单的解释机制.

### 2.1.3 优点

LambdaRank 不仅适用于 NN 模型, 也适用于其它模型; LambdaRank 不仅适用于 ranking, 也能被扩展到任何 non-smooth 和 multivariate 的 cost functions.

## 2.2 简介

背景:

许多 inference tasks 中, 评估一个系统的最终质量的 cost function 并不是训练期间使用的 loss function. 例如, 对一个SVM的二分类模型, 我们可能会用 error rate 来评估其质量, 而训练时的 cost function 大多使用 MSE 或 cross entropy.

因此在机器学习任务中, 通常有2种 cost functions: 1) the desired cost; 2) 训练过程中使用的 cost. 通常, 我们称前者为 'target' cost, 而后者为 'optimization' cost. 而 optimization cost 起2方面的作用: 1) 使优化任务能够解决; 2) 能够很好地近似 desired cost.

事实上, 两种 cost 之间的不匹配不仅在分类领域存在, 在 IR 领域也非常严重. 如 IR 领域有许多常用的 quality measures(本文中含义与 desired cost 相同) 仅依赖于文档的排序结果和标签的相关性. 而 target cost 通常是多个 query 的平均值. 这种 target costs 给 ML 带来的挑战是: 它们要么是平的(即相对于 model score 的梯度为0), 要么是不连续的. 因此这种 target and optimization costs 之间的不匹配很可能带来算法的准确率问题.

创新点:

不同于创建 smooth versions of the cost function(内在的 'sort' 本质给这种方法带来很大挑战), 我们的方法通过定义每个 item 排序后的 virtual gradient 绕过了排序问题. 而且简单且具有泛化性, 能应用于任意 target cost function 中.

## 2.3 IR 领域常见的 Quality Measures

介绍了常见的评价指标, 包括 MAP/MRR(Mean reciprocal rank)/WTA(Winner Take All)/Pair-wise Correct/NDCG, 而且 NDCG 非常适合用在 IR 中, 本文以此为评价指标.

$$
    \mathcal{N}_i \equiv N_i \sum^L_{j=1} (2^{r(j)} - 1) / log(1+j)
$$

其中 L 为截断系数, 即从 L 个开始排序的结果开始不再计算 NDCG, 这可能与 lambdarank 中的 lambdarank_truncation_level 参数有关.

## 2.4 之前的工作

nothing special, skip this part.

## 2.5 LambdaRank 方法

本节主要介绍了 LambdaRank 的思想: 对于原来 optimisation cost, 在此基础上乘以一个基于规则的虚拟梯度函数 lambda, 从而得到最终每次迭代的梯度值.

这个主要思想的直觉想法是: 首先, 对于给定 query 的 documents 的得分排序后, 找到一个我们希望如何更改文档的排名顺序的规则, 要比构建一个通用且平滑的优化函数要容易得多, 我们只需要为给定的排序指定规则, 对感兴趣的特定点(个人理解为有相关的 document的位置)定义隐式的 cost function 的梯度; 其次, 基于规则的函数可以编码我们对算法的期待(即关注相关且排名靠前的 documents 并花少量代价让它排到最前面, 而非花大量代价将后面的文档排到前面来, 参考本节第1段的例子). 用公式表示即为:

$$
\frac{\delta{C}}{\delta{s_j}} = - \lambda_j(s_1,l_1,\dots,s_{n_i},l_{n_i})
$$

对于第 i 个查询, doc 的排序位置 j, 如果 $\lambda$ 为正, 则应该提高排名来降低 C. 可见 $\lambda$ 取决于根据 score function 排序后的位置.

现在需要确定 $\lambda$, 使得:
1. 对于 $\lambda$, 是否存在以及如何找到 C 使得上述梯度式子成立;
2. C 应该是 convex.

根据旁加莱定义(Poincare Lemma): if $S \subset R^n$ is an open set that is star-shaped with respect to the origin, then every closed form on S is exact; $\lambda$ 只需要满足:
$$
    \begin{gather}
        \sum_j \lambda_j dx^j =& 0 \\
        \frac{\delta{\lambda_j}} {\delta{s_k}} =& \frac{\delta{\lambda_k}} {\delta{s_j}}, \forall{j,k} \in {\{1,2,\dots,n_i}\}
    \end{gather}
$$

最后强调 LambdaRank 提供了规则来隐式地利用给定的排序的 documents 的 cost, 而且这种规则非常容易想到.

## 2.6 LambdaRank 应用到 RankNet

本节参考自文献5, 但内容较为简单, 只有大概明确, 详细内容参考文献2.

给定一对文档, cost 为两者的乘积, 而 RankNet Cost 如下式. 该计算为 pairwise 的, 只基于这对文档的信息. 无论之前排序是否正确, 只要 $s_j - s_i$ 增大(如果 $s_j < s_i$), cost 就会减小.
$$
    \begin{gather}
        C^R_{i,j} =& s_j - s_i + log{(1 + e^{s_i - s_j})} \\
        \frac {\delta{C_{ij}}} {\delta{o_{ij}}} =& \frac {-1} {1 + e^{o_{ij}}}
    \end{gather}
$$

假设两个文档排序互换能得到的 NDGC 增益为 $\Delta{NDCG}$, 计算如下. 该计算基于 query 的整个返回结果计算.
$$
    \begin{gather}
        N_i =& n_i \sum^L_{j=1} (2^{rel(j)} - 1) / log{(1 + j)} \\
        \Delta{N_i} = & n_i (2^{l_i} - 2 ^{l_j}) (\frac {1} {log{(1+i)}} - \frac {1} {log{(1 + j)}})
    \end{gather}
$$

将两个 gradient 相乘得到 $\lambda$:
$$
\lambda = N(\frac {1} {1 + e^{s_i - s_j}})(2^{l_i} - 2^{l_j})(\frac {1} {log(1+i)} - \frac {1} {log(1+j)})
$$

# 参考文献

1. [lambdarank 论文](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/lambdarank.pdf)
2. [LambdaMART 系列论文 overview](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.180.634&rep=rep1&type=pdf)
3. [lightgbm 官网的参数介绍](https://lightgbm.readthedocs.io/en/v3.3.2/Parameters.html)
4. [浅谈Learning to Rank中的RankNet和LambdaRank算法](https://zhuanlan.zhihu.com/p/68682607)
5. [LambdaRank 博客](http://iccm.cc/learning_to_rank_with_nonsmooth_cost_functions/)