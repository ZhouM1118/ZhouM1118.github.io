---
layout: post
title:  "机器阅读理解Attention-over-Attention模型"
date:   2018-05-18
desc: "机器阅读理解Attention-over-Attention模型"
keywords: "Attention-over-Attention, 机器阅读理解, 深度学习"
categories: [Deeplearning]
tags: [Attention-over-Attention, 机器阅读理解, 深度学习]
icon: icon-html
---
# **0 前言**

Attention-over-Attention模型（AOA Reader模型）是科大讯飞和哈工大在2017ACL会议上的《Attention-over-Attention Neural Networks for Reading Comprehension》（[论文地址](https://arxiv.org/abs/1607.04423)）联合提出的。科大讯飞和哈工大在2016ACL会议上发表的另一篇论文《Consensus Attention-based Neural Networks for Chinese Reading Comprehension》提出了CAS Reader模型（在博文[《HFL-RC：科大讯飞填空式机器阅读理解数据集》](https://zhoum1118.github.io/deeplearning/2018/04/26/HFL-RC-%E7%A7%91%E5%A4%A7%E8%AE%AF%E9%A3%9E%E5%A1%AB%E7%A9%BA%E5%BC%8F%E6%9C%BA%E5%99%A8%E9%98%85%E8%AF%BB%E7%90%86%E8%A7%A3%E6%95%B0%E6%8D%AE%E9%9B%86.html)有详细介绍），**AOA Reader正是基于CAS Reader进行改进的，两个模型都属于二维匹配模型**，CAS Reader仅按照列的方式进行Attention计算，但AOA Reader结合了按照列和按照行的方式进行Attention计算，同时使用了二次验证的方法对AOA Reader模型计算出的答案进行再次验证。下面我们将详细分析下AOA Reader模型的实现过程。

# **1 数据集及任务分析**

数据集采用完形填空式的数据集，包含CNN&Daily Mail以及儿童故事（Children’s Book Test，CBT），数据集介绍在[《ASReader：一个经典的机器阅读理解深度学习模型》](https://zhoum1118.github.io/deeplearning/2018/05/09/ASReader-%E4%B8%80%E4%B8%AA%E7%BB%8F%E5%85%B8%E7%9A%84%E6%9C%BA%E5%99%A8%E9%98%85%E8%AF%BB%E7%90%86%E8%A7%A3%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0%E6%A8%A1%E5%9E%8B.html)中有，这里就不再赘述。

一个完形填空式阅读理解样本可描述为一个三元组：`<D, Q, A>`，其中`D`代表文档（document），`Q`代表问题（query），`A`则代表问题的答案（Answer），**答案是文档中某个词，答案词的类型为固定搭配中的介词或命名实体**。

# **2 AOA Reader模型**

图1 展示了AOA Reader模型的框架图

<img title="机器阅读理解Attention-over-Attention模型_" 图片1="" src="https://img.mukewang.com/5afe86a5000131b618741214.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图1 AOA Reader模型的框架图</p>

## **2.1 Contextual Embedding**

首先将文档`D`和问题`Q`转化为`one-hot向量`，然后将`one-hot向量`输入embedding层，这里的**文档嵌入层和问题嵌入层的权值矩阵共享**，通过共享词嵌入，文档和问题都可以参与嵌入的学习过程，然后使用双向GRU分别对文档和问题进行编码，**文档和问题的编码都拼接正向和反向GRU的隐藏层输出，这时编码得到的文档和问题词向量都包含了上下文信息**。计算过程如下所示：

<img title="机器阅读理解Attention-over-Attention模型_" 图片1="" src="https://img.mukewang.com/5afe86c10001e66906480350.png" alt="图片描述" style="display:block; margin:auto; width:30%">

文档的`Contextual Embedding`表示为`h_doc`，维度为`|D| * 2d`，问题的`Contextual Embedding`表示为`h_query`，维度为`|Q| * 2d`，`d`为GRU的节点数。

## **2 .2 Pair-wise Matching Score**

将`h_doc`和`h_query`进行**点积计算**得到成对匹配矩阵`（pair-wise matching matrix）`，表示如下：

<img title="机器阅读理解Attention-over-Attention模型_" 图片1="" src="https://img.mukewang.com/5afe86de0001625505720102.png" alt="图片描述" style="display:block; margin:auto; width:30%">

`M(i, j)`代表文档中第`i`个单词的`Contextual Embedding`与问题中第`j`个单词的`Contextual Embedding`的点积之和，`M(i, j)`这个矩阵即图1中点积之后得到的矩阵，横向表示问题，纵向表示文档，故`M(i, j)`的维度为`|D| * |Q|`。

## **2.3 Individual Attentions**

这一步和[《HFL-RC：科大讯飞填空式机器阅读理解数据集》](https://zhoum1118.github.io/deeplearning/2018/04/26/HFL-RC-%E7%A7%91%E5%A4%A7%E8%AE%AF%E9%A3%9E%E5%A1%AB%E7%A9%BA%E5%BC%8F%E6%9C%BA%E5%99%A8%E9%98%85%E8%AF%BB%E7%90%86%E8%A7%A3%E6%95%B0%E6%8D%AE%E9%9B%86.html)中的CAS Reader模型一样：**在二维模型中是按列进行计算的，计算的是文档中每个单词对问题中的某个单词的重要程度（即注意力）**，最后形成一个文档级别的注意力分布`a(t)`，问题中有多少个单词就有多少个这样的分布，文档中的某些单词都对问题中的单词A有注意力，同时也对问题中的单词B有注意力，那么将文档中某个单词对问题中单词的注意力进行汇总，汇总的方式有取平均数，累加，最大值等。

计算过程如下：

<img title="机器阅读理解Attention-over-Attention模型_" 图片1="" src="https://img.mukewang.com/5afe86f700015efb07580152.png" alt="图片描述" style="display:block; margin:auto; width:35%">

**说人话就是：**你在回答问题的时候，每阅读问题的一个单词，心中就会有一个文档中的单词对这个问题单词的相关性分布，当你读完整个问题之后，将文档中每个单词在这些分布中的值进行汇总，如果说是取最大值，那么意味着将文档中每个单词在这些分布中的最大值取出作为该单词对问题的相关性，（文档中某单词可能出现多次，那么会有多个相关性值，将其进行累加再作为该单词对问题的相关性），最后取相关性最高的词作为问题的答案。

其中，

```
a(1) = softmax(M(1, 1), …, M(|D|, 1))；
a(2) = softmax(M(1, 2), …, M(|D|, 2))
```

那么`a=[a(1), a(2),…, a(|Q|)]`，也就是`a`的维度为`|D| * |Q|`。

在CAS Reader模型中，对于矩阵`a`，**进行如下三种运算之一**：

<img title="机器阅读理解Attention-over-Attention模型_" 图片1="" src="https://img.mukewang.com/5afe870a000192c309460322.png" alt="图片描述" style="display:block; margin:auto; width:40%">

其中，`m`为问题的单词个数，也就是说如果`mode=sum`，则对矩阵`a`按列进行累加，最后**累加值最大对应的文档的单词即为答案**。

**但AOA Reader在得到`a`矩阵之后，并没有进行以上运算而是再对二维模型进行了按行计算。**

## **2.4 Attention-over-Attention**

**在二维模型中按行进行计算，计算的是问题中的每个单词对文档中某个单词的重要程度（即注意力）**，形成一个问题级别的注意力分布`B(t)`，文档中有多少个单词就有多少个这样的分布，然后对这些分布进行累加并求平均得到`B`，最后将`Individual Attentions`中得到的`a`与`Attention-over-Attention`得到的`B`进行**点积计算**得到`s`，这里的`a`的维度为`|D| * |Q|`，`B`的维度为`|Q| * 1`。计算过程如下：

<img title="机器阅读理解Attention-over-Attention模型_" 图片1="" src="https://img.mukewang.com/5afe871e000112df07520088.png" alt="图片描述" style="display:block; margin:auto; width:40%">

<img title="机器阅读理解Attention-over-Attention模型_" 图片1="" src="https://img.mukewang.com/5afe8728000125ff03360180.png" alt="图片描述" style="display:block; margin:auto; width:20%">

**说人话就是，**我们在看问题的时候，并不是问题的每个单词我们都需要用到解题中，即问题中的单词的重要性是不一样的，在一步我们主要分析问题中每个单词的贡献，先定位贡献最大的单词，然后再在文档中定位和这个贡献最大的单词相关性最高的词作为问题的答案。

## **2.5 Final Predictions**

最后和CAS Reader模型一样计算单词`w`是答案的条件概率，文档`D`的单词组成单词空间`V`，单词`w`可能在单词空间`V`中出现了多次，其出现的位置i组成一个集合`I(w, D)`，对每个单词`w`，我们**通过计算它的注意力值并求和得到单词`w`是答案的条件概率**，计算公式如下所示：

<img title="机器阅读理解Attention-over-Attention模型_" 图片1="" src="https://img.mukewang.com/5afe8738000118d706460158.png" alt="图片描述" style="display:block; margin:auto; width:30%">

# **3 二次验证：N-best Re-ranking Strategy**

对于完形填空式的阅读理解数据集，可将2中构建的神经网络模型预测的候选答案带入到问题的空白处进行二次验证，选取前`n`个最有可能的候选答案组成`N-best list`，将候选答案填入问题中，并提取以下三种特征：

 1. Global N-gram LM：基于训练集中所有的document训练8-gram模型，判断候选答案所在句子的合理性；
 2. Local N-gram LM：基于问题所对应的的document训练8-gram模型；
 3. Word-class LM：和Global N-gram LM类似，基于训练集中所有的document，但Word-class
LM使用聚类方法对候选答案进行打分，这里使用了mkcls工具产生1000个单词类别。

采用`K-best MIRA`算法对三种特征进行权值训练，最后累加权值，选择损失函数最小的值作为最终的正确答案。

# **4 实验设置及结果**

## **4.1 实验设置**

 - 嵌入层（使用未预训练词向量）权值初始化：[−0.05, 0.05]；
 - l2正则化：0.0001；
 - dropout：0.1；
 - 优化函数：Adam；
 - 学习率初始化：0.001；
 - gradient clip：5；
 - batch size：32；

根据不同的语料库，嵌入层和GRU隐藏层的节点数如图2所示。

<img title="机器阅读理解Attention-over-Attention模型_" 图片1="" src="https://img.mukewang.com/5afe874700011c3e09080318.png" alt="图片描述" style="display:block; margin:auto; width:50%">

<p style="text-align:center">图2 不同的语料库，嵌入层和GRU隐藏层的节点数</p>

## **4.2 实验结果**

<img title="机器阅读理解Attention-over-Attention模型_" 图片1="" src="https://img.mukewang.com/5afe875400015dd917361228.png" alt="图片描述" style="display:block; margin:auto; width:70%">