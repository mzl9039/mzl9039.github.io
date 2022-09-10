---
layout:     post
title:      论文阅读-Sampling-Bias-Corrected Neural Modeling for Large Corpus Item Recommendations
subtitle:
date:       2022-09-05 09:52:00
author:     mzl
catalog:    true
tags:
    - retrieval
    - sampling bias
    - IR
    - search
---

# 背景介绍

本文是 google/youtube 2019 年在 RecSys 发表的论文。具体介绍如何估计采样偏差并对其偏差进行纠正。因为作者采用了 online hard negative sampling 的方法训练模型，这种方式的采样实际上鼓励热门物品，压制冷门物品，因此对偏态分布的数据存在采样上的偏差，作者提出了对采集偏差进行估计并修改的方法。

# Abstract

许多推荐系统的 corpus 非常大，这时处理数据稀疏性和幂律分布的方法是学习其 content features，如双塔模型中的 item 塔。训练这种双塔模型的一种方法是在 mini-batch update 时对 batch 内负采样的样本优化其 loss(指 triplet loss)，但这种采样方式对高度偏态分布存在采样偏差从而影响模型性能。作者提出从流式数据中估计 item 分布的新算法，可用于纠正采样的偏差，并在 Youtube 的推荐系统中部署和使用，线上 A/B test 中取得不错的效果。

# 1.简介

基本的推荐系统历史介绍，工业界召回碰到的问题(数据模型非常庞大，从用户端收集的数据稀疏导致对长尾部分方差大)。推荐系统的一些主流方案（如 matrix factorization）的优点及不足。近年来 DNN 用于推荐系统的发展及双塔用于推荐系统的历史(来自多分类方法)。最后提到，前人在 MLP 方法中用到了重要性采样和减小偏差的方法引发作者提出了通过频率估计及修正偏差的方法。

主要贡献：
1. 流式频率估计。提出一种基于数据流对 item frequency 的估计方法，用于估计 item frequency 受字典和分布偏移(distribution shifts)的影响。
2. 建模框架。提出通用的建模框架，尤其是将估计的频率用于交叉熵损失中以修正 in-batch sampling 带来的采样偏差。
3. Youtude 推荐。介绍如何将模型框架部署到 Youtude 的大型召回系统，包括训练、索引及服务。
4. 线下和线上实验。线下验证了偏差估计的正确性，线上效果提升。

# 2.相关工作

## 2.1 上下文感知和神经网络推荐

利用用户和 item 的内容特征是提升泛化性和缓解冷启动问题的关键。有一系列将内容特征应用于经典的 MF 的研究，主要是泛化 MF 模型以利用内容特征，如 SVDFeature[<sup>2</sup>](#ref-2), 这些模型可以捕获到特征的 bi-linear or second-order 交叉关系。近些年来，DNN 的发展极大的提升了推荐的准确率，因为其非线性的能力可以捕获更复杂的特征交叉关系。He[<sup>3</sup>](#ref-3) 基于协同过滤(CF)提出 Neural CF 来建模 item-user 交叉。

有一系列工作使用 RNN 模型提取序列特征用于推荐。另外，除了学习单独的 user 和 item 表征外，有一系列工作关注综合学习用于排序。最后 Cheng[<sup>4</sup>](#ref-4) 提出 Wide&Deep 框架来联合学习 wide linear model 和 deep nerual networks。

## 2.2 Extreme Classification

当有 millions of labels 用于多分类时，softmax 是最常用的 loss。适用性采样减少标签数量是一个常见方法，还有很多采用 tree-based label structure (如分层 softmax) 被用于分类模型从而减少 inferance 耗时。但这需要预定义标签而不适用于有广泛的、变化的输入的情况(?什么意思)。

## 2.3 双塔

双塔模型在自然语言任务是非常流行，但和自然语言任务相比，作者关注在非常庞大的 corpus 上(如 Youtube)进行推荐。通过线上实验，作者发现建模 item 频率有助于提升 retrieval 的准确率。

# 3.模型框架

考虑常规的推荐问题，即给定 query 和 item 集合，分别由特征向量 $\{x_i\}^N_{i=1}$ 和 $\{y_j\}^M_{j=1}$ 表示，且 $x_i \in \mathcal{X},\ y_j \in \mathcal{Y}$ 都是由一系列 sparse 和 dense 特征组成的，可能有非常高的维度。目标是为每个 query 召回一个 item 集合。在个性化召回场景下，假定 user 及 context 信息在特征 $x_i$ 中。

作者旨在构建1个双塔模型，包含2个 embedding 函数用于将 query 及 item 特征映射到 k 维空间中：$u: \mathcal{X} \times \mathbb{R}^d \rightarrow \mathbb{R}^k, v: \mathcal{Y} \times \mathbb{R}^f \rightarrow \mathbb{R}^k$, 其中 $u, v$ 表示两个映射函数; 模型是参数为 $\theta$, 模型的输出是2个模型的内积：

$$
s(x,y) = \langle u(x, \theta), v(y, \theta) \rangle
$$

目标就是从训练集 T 中学习模型参数 $\theta$, 表示为：

$$
\mathcal{T} := \{ (x_i, y_i, r_i) \}^T_{i=1}
$$

其中 $(x_i, y_i)$ 是 query $x_i$ 和 item $y_i$ 对，而 $r_i \in{\mathbb{R}}$ 是 query-item 对的 reward。(<font color="red">这里 reword 的存在是本文产生的根本原因，因为这个的分布与线上的分布需要相同才行，否则会严重影响模型的结果</font>)。

直观上看，召回问题可以视为连续 reward 下的多分数问题(即 reward 分区间作为类别)，此时每个 positive pair 的 reword 是 $r_i = 1$。实际的推荐系统中，可以将 $r_i$ 进行扩展，表示用户的多种活动参与程度，如用户浏览文章的耗时等。给定 query $x$ 时，从召回集合 M 中选择候选 $y_i \in \{ y_i\}^M_{j=1}$ 是基于如下的 softmax 函数：

$$
\begin{gather}
    \mathcal{P(y|x; \theta) = \frac {e^{s(x,y)}} {\sum_{j \in [M]} e^{s(x,y_j)}}}
\end{gather}
$$

为进一步利用 reword $r_i$, 可以考虑如下的加权 log-likelihood 作为 loss 函数

$$
\begin{gather}
    L_T(\theta) := - \frac {1} {T} \sum_{i \in [T]} r_i \centerdot log(\mathcal{P}(y_i|x_i; \theta))
\end{gather}
$$

当召回集合 M 非常大时，计算上述 loss 是不现实的，可行的方案是使用一个子集来构建这样的分区函数，这里采用流数据。然而，不同于 nlp 从固定大小的 corpus 中采样，这里由于采用 online 的方式进行负采样(即 in-batch 内进行负采样)。即给定一个 mini-batch $B = \{ (x_i, y_i, r_i) \}^B_{i=1}$，对任意 $i \in B$，batch softmax 计算方式为

$$
\begin{gather}
    \mathcal{P_B(y|x_i; \theta) = \frac {e^{s(x_i,y_i)}} {\sum_{j \in [B]} e^{s(x_i,y_j)}}}
\end{gather}
$$

<font color="red">由于 in-batch 是从符合幂律分布的目标应用中采样的，因此 Equation (3) 对 full softmax 带来的世大的采样偏差：由于热门 item 在 in-batch 中概率较高，因此其作为负样本被过度 penalized。</font> 受 logQ 纠正采样的 softmax 模型 [<sup>5</sup>](#ref-5) 的启发，将点积计算修正为：

$$
\begin{gather}
    s^c(x_i, y_i) = s(x_i, y_i) - log(p_j)
\end{gather}
$$

这里 $p_j$ 表示 item j 在任意 batch 内的采样概率。基于上述的修正，修正后的 batch softmax 更新为：

$$
\begin{gather}
    \mathcal{P^c_B(y|x_i; \theta) = \frac {e^{s^c(x_i,y_i)}} {e^{s^c(x_i,y_i)} + \sum_{j \in [B], j\neq i} e^{s^c(x_i,y_j)}}}
\end{gather}
$$

将上式嵌入 Equation (2) 得到 batch loss function 如下：

$$
\begin{gather}
    L_B(\theta) := - \frac {1} {B} \sum_{i \in [B]} r_i \centerdot log(\mathcal{P^c_B}(y_i|x_i; \theta))
\end{gather}
$$

接下来就是利用 SGD 并应用学习率 $\gamma$ 更新模型参数：

$$
\begin{gather}
    \theta \leftarrow \theta - \gamma \centerdot \nabla{L_B(\theta)}
\end{gather}
$$

注意到，Equation (5) 不需要固定的 query 或 item 集合，因此可以用于流式数据。Algrithm (1) 表示作者提出的上述方法。

![Algrithm (1): Training Algorithm](/styles/img/sampling-bias-correct_training_algorithm.png)

另外，文献[<sup>6,</sup>](#ref-6)[<sup>7,</sup>](#ref-7)[<sup>8</sup>](#ref-8)可以帮助近似 maximum inner product search (MIPS) 问题。具体地，即通过量化[<sup>9</sup>](#ref-9)和端到端学习稀疏和点积量化器[<sup>10</sup>](#ref-10)建立高维数据的 dense 表征(？这里的两篇文献貌似是 ANN 相关内容，如 Faiss/hnsw 等的原理)。

*Normalization and Temperature* (<font color="red">这里介绍了2个 trick</font>): 1) 通过对 embedding 进行规范化能提升召回质量，有助于训练，即 $u(x, \theta) \leftarrow u(x, \theta) / \lVert u(x, \theta) \rVert_2, \ v(x, \theta) \leftarrow v(x, \theta) / \lVert v(x, \theta) \rVert_2$; 2) 将 temperature 超参数 $\tau$ 加到内积上能加强预测结果，即 $s(x,y) = \langle u(x, \theta), v(y, \theta) \rangle / \tau$。

# 4.Streaming Frequency Estimation

*频率估计方法要求是分布式地*：考虑 a stream of 随机 batches，每个 batch 是一个 item 集合。核心问题是估计每个 batch 中 hitting 每个 item 的概率(这里的 hitting 是什么意思?)。关键的设计原则是要有一个充分地分布式的算法来应用分布式训练任务。

*使用 global step 进行分布式地频率估计*：不论是单机还是分布式训练，一个全局唯一的 step 被关联到每个 sampled batch，并代表被某个 trainer 消费的 batch 内的数据。在分布式环境中，这个 step 通过参数服务器被同步到多个 workers 中，可能考虑利用这个 global step 将某个 item 的频率 $p$ 转为估计的 $\sigma$, 该值表示连续两次命中该 item 的平均步数。例如，某个 item 每 50 steps 被采样一次，意味着 $p=0.02$。使用 global steps 的好处有2个：1) 通过读写 global steps，多个 workers 被隐式地同步了；2) 估计值 $\sigma$ 可以通过移动平均地方式更新，而这种方式对分布的变化是自适应的。

*移动平均的方式更新估计值 $\sigma$*: 使用数组 array 记录采样的 item id(由于 corpus 中 item 太多，记录每个 item 不现实，因此使用了 hash 函数将 item id 或其特征值映射到数组中)。这里使用2个数组 A 和 B 分别记录上一次采样到该 item 的步数和 item 的估计值 $\sigma$，两个数组的大小均为 H，hash 函数记为 $h(y)$，就可以用数组 A 帮助更新数组 B。当 step t 时又采样到 item y, 则按下面的方式更新 B，然后将 t 赋给 $A[h(y)]$，如 Algorithm (2) 所示。

$$
\begin{gather}
    B[h(y)] \leftarrow (1 - \alpha) * B[h(y)] + \alpha * (t - A[h(y)])
\end{gather}
$$

![Algrithm (2): Streaming Frequency Estimation](/styles/img/sampling-bias-correct_freq-estimation.png)

*估计值 $\sigma$ 的原理*：如果用随机变量 $\Delta$ 表示分布中某 item 连续两次采样采中的步数间隔，则 $\sigma = E(\Delta)$。这里目标是估计 a stream of 采样 item 的 $\sigma$，因此当 item 在 step t 时再次在某个 batch 中被采样，则 $t - A[h(y)]$ 就是一个 $\Delta$ 的采样值，则通过上述的更新步骤就可以通过 SGD 以固定的学习率 $\alpha$ 学到该随机变量的平均值。

上述方法是 online 估计频率的方法，该方法的 bias 和 vairance 的证明这里跳过不写了。注意到，证明公式表明，估计值初始化为 $\sigma_0 = \sigma / (1 - \alpha)$。另外 item 的采样概率 $\hat{p} = 1/B[h(y)]$。

*Multiple Hashing*: 上面提到，将 item id 或其它特征值通过 hash 函数映射到长度为 H 的数组中，这个步骤可能导致 hash 冲突，因此这里将 hash 函数修改为利用多个 hash 函数来缓解由于冲突带来的 over-estimation 问题，如 Algorithm (3) 所示。

![Algorithm (3): Improved Frequency Estimation with Multiple Arrays](/styles/img/sampling-bias-correct_improved-freq-estimation.png)

# 5.Youtube 的 Neural Retrieval System

系统是常规的召回+排序的结构，这里关注召回阶段的数据、模型架构、训练和服务。

## 5.1 模型 overview

模型架构如下图所示。就是常规的双塔。

![model architecture](/styles/img/sampling-bias-correct_model-architecture.png)

*训练标签*: 点击为正样本，并使用 reword 表示用户对当前 video 的感兴趣程度，如用户只看了该视频的很短时间则 $r_i = 0$，如果用户看完了则 $r_i = 1$。

特征包括：视频特征，用户特征等，跳过。

## 5.2 序列训练

根据每天的数据重新生成新一天的数据集，并进行分布式模型训练，以纠正每天的 distribution shift，并在训练时采用上节的频率估计方法纠正采样偏差。

## 5.3 索引和模型服务

跳过。

# 6.实验

跳过。

# 7.总结

提出新的方法来估计 item frequency，并为提出的方法提出了序列训练策略使得线上分布式训练可以集成。理论分析和模拟实验证明了它的正确性和有效性，基于 Wiki 的链接预测证明有效，而 Youtube 的线上实验显示其在提升用户参与和候选视频召回上都有明显效果。

# 参考文献

<div id="ref-1"></div>

[1] [Sampling-Bias-Corrected Neural Modeling for Large Corpus](https://storage.googleapis.com/pub-tools-public-publication-data/pdf/6c8a86c981a62b0126a11896b7f6ae0dae4c3566.pdf)

<div id="ref-2"></div>

[2] [Chen tianqi, SVDFeature: a toolkit for feature-based collaborative filtering](https://www.jmlr.org/papers/volume13/chen12a/chen12a.pdf)

<div id="ref-3"></div>

[3] [Neural Collaborative Filtering](https://arxiv.org/pdf/1708.05031.pdf)

<div id="ref-4"></div>

[4] [Wide Deep Learning for Recommender Systems](https://dl.acm.org/doi/pdf/10.1145/2988450.2988454)

<div id="ref-5"></div>

[5] [Adaptive importance sampling to accelerate training of a neural probabilistic language model](https://infoscience.epfl.ch/record/82914/files/rr-03-35.pdf)

<div id="ref-6"></div>

[6] [Near-optimal Hashing Algorithms for Approximate Nearest Neighbor in High Dimensions](http://people.csail.mit.edu/indyk/p117-andoni.pdf)

<div id="ref-7"></div>

[7] [Approximating matrix multiplication for pattern recognition tasks](https://www.academia.edu/download/40244212/Approximating_Matrix_Multiplication_for_20151121-13854-18rrzhq.pdf)

<div id="ref-8"></div>

[8] [Efficient Retrieval of Recommendations in a Matrix Factorization Framework](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.401.390&rep=rep1&type=pdf)

<div id="ref-9"></div>

[9] [Quantization based Fast Inner Product Search](http://proceedings.mlr.press/v51/guo16a.pdf)

<div id="ref-10"></div>

[10] [Multiscale Quantization for Fast Similarity Search](https://proceedings.neurips.cc/paper/2017/file/b6617980ce90f637e68c3ebe8b9be745-Paper.pdf)