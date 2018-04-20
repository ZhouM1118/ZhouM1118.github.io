---
layout: post
title:  "CNN&Dailymail：Teaching Machines to Read and Comprehend"
date:   2018-04-15
desc: "CNN&Dailymail：Teaching Machines to Read and Comprehend"
keywords: "机器阅读理解数据集, CNN&Dailymail"
categories: [Deeplearning]
tags: [机器阅读理解数据集, CNN&Dailymail]
icon: icon-html
---

**0 CNN&Dailymail语料库**
------------------

**0.0 背景**

教会机器能够阅读并理解人类文章是人工智能的一个前沿方向，也是机器更好地为人类服务必须具备的一个能力。传统机器阅读理解方法大多为无监督学习，即**采用模板或句法分析器从上下文文档中提取实体关系元组（谓语及主、宾语），并人工设计规则匹配文档信息，保存以供查询**。这种简单且严重依赖人工经验的方式并不能取得很好的机器阅读理解效果，但有监督学习方法需要大量带标签的阅读理解数据，语料库的极度缺乏也是影响机器阅读理解方法发展的重要原因之一。

虽然带标签的阅读理解数据获取困难，但有一些学者开始构造**人造叙述性文章以及问题数据集**，大量的人造数据并以此为语料库构建机器阅读理解的神经网络模型取得了较好的效果，但实践证明，基于人造数据训练得到的网络模型不一定适用于真实应用场景中，因为模拟数据永远**无法捕捉到自然语言的复杂性、丰富性和噪点**。

**0.1 CNN&Dailymail**

于是，Hermann等人于2015年在[《Teaching Machines to Read and Comprehend》](https://arxiv.org/abs/1506.03340)一文中发布CNN&Dailymail数据集。[数据集下载](https://github.com/deepmind/rc-data)。Hermann从美国有线新闻网（CNN）和每日邮报网中收集了大约一百万条新闻数据作为机器阅读理解语料库，并通过实体检测等方法**将总结和解释性的句子转化为[背景, 问题, 答案]三元组**。图1展示的是CNN&Dailymail语料库的统计信息。

<img title="CNN&amp;Dailymail：TeachingMachinestoReadandComprehend_" 图片1="" src="https://img.mukewang.com/5ad96e4b00018cf008900430.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图1 CNN&Dailymail语料库的统计信息</p>

CNN&Dailymail语料库由2007年4月-2015年4月的CNN新闻文章和2010年6月-2015年4月的每日邮报网新闻文章组成，语料库剔除了单篇超过2000个字的文章和问题答案不在原文出现的文章。

**0.2 命名实体替换**

Hermann将机器阅读理解任务看作是一个**有监督学习问题**，并尝试求出语料库中每个样本的答案（answer）的条件概率`p(a|c,q)`，`c`表示上下文文档，`q`表示问题，`a`则表示该问题的答案。**Hermann尝试剔除词组共现统计对机器阅读理解的影响，让网络模型更关注从上下文挖掘实体的语义关系**，故而Hermann将语料库中文章样本的命名实体都进行了替换，且在训练与测试过程中，命名实体替换后都会进行随机排序，防止在训练过程中网络模型过渡关注替换后的命名实体，而忽略去理解问题以及上下文信息，提高模型泛化能力。图2展示了原始版本与命名实体替换后的匿名版本。

<img title="CNN&amp;Dailymail：TeachingMachinestoReadandComprehend_" 图片1="" src="https://img.mukewang.com/5ad96e7200010b5015340678.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图2 原始版本与命名实体替换后的匿名版本对比</p>

**为什么要进行命名实体替换？**
很显然，人在原始版本和匿名版本的文章中都很容易能回答问题，但对于匿名版本的问题，人必须要去阅读原文才能回答问题，而对于原始版本，如果人在不阅读原文仅通过阅读分析问题的情况下也是有可能回答问题的，因为人可能根据实体信息猜出答案，如果是这样则丢失了机器阅读理解的本质。

从这里也能够看出**CNN&Dailymail语料库的一些特点**：答案是某种实体对象；答案一定出现在原文中，所以对于需要让机器理解并回答推理性的问题，CNN&Dailymail语料库力不从心。

**1 模型**
----

**1.0 符号匹配模型**

（1）**框架语义分析模型(Frame-semantic)**，模型通过识别句子谓语以及它们的主语和宾语，匹配类似**“谁对谁做了什么事情”**的框架来获取信息。

分别从问题`q`和上下文文档`d`中提取实体与谓词的三元组，三元组表示为`(e1，V，e2)`，并使用事先定义好的多个规则来匹配问题的答案，如果在一个规则下出现多个答案，则随机选择一个。图3展示了规则定义以及举例分析。

<img title="CNN&amp;Dailymail：TeachingMachinestoReadandComprehend_" 图片1="" src="https://img.mukewang.com/5ad96ea00001144914640334.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图3 框架语义分析模型的规则定义以及举例分析</p>

（2）**单词距离基准法(Word distance)**，将完形填空式问题中的答案占位符与上下文文档中的每个可能的实体对比，并计算问题与指定实体上下文之间的距离，选取距离最小的实体对象作为问题答案。

**1.1 神经网络模型**

**1.1.1 Deep LSTM Reader**

先将原文文档document一次一个单词token输入到一个两层LSTM中，然后将问题query一次一个单词输入到两层LSTM，即模型的输入不仅包含原文文档，还包含问题，原文与问题用分隔符（`|||`）划分开，也可以反过来。即使用两个LSTM来编码encode `document ||| query` 或 `query ||| document`，图4展示了双层LSTM对`query ||| document`的编码过程，通过先后将query和document输入双层LSTM中，拼接后通过一个`g(q,d)`函数得到`query ||| document`的向量表示，然后用得到的表示做分类模型的输入。

<img title="CNN&amp;Dailymail：TeachingMachinestoReadandComprehend_" 图片1="" src="https://img.mukewang.com/5ad96ebd00013d8f06340284.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图4 双层LSTM对query ||| document的编码过程</p>

通过一个`g(q,d)`函数得到`query ||| document`的向量表示作为输入`X(t)`，然后输入至一个双层LSTM得到答案，计算演变过程如图5所示。

<img title="CNN&amp;Dailymail：TeachingMachinestoReadandComprehend_" 图片1="" src="https://img.mukewang.com/5ad96ed100017e4313260404.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图 5 双层LSTM计算演变过程</p>

其中，`||`表示拼接两个向量，`h(t, k)`表示`t`时刻在`k`层隐藏层的状态，`i, f, o`分别表示输入门、遗忘门以及输出门的状态。

**1.1.2 Attentive Reader**

Deep LSTM Reader试图挖掘长距离依赖信息，但这一目的却受限于固定宽度的隐藏层。受机器翻译与图像识别的启发，Attentive Reader采用注意力机制来构建token级别的网络模型。图6 展示了Attentive Reader中query和document的编码过程。

<img title="CNN&amp;Dailymail：TeachingMachinestoReadandComprehend_" 图片1="" src="https://img.mukewang.com/5ad96ee700017e9707140408.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图6 Attentive Reader中query和document的编码过程</p>

首先使用双向LSTM层分别对文档和问题进行编码，对于长度为`|q|`的query编码，拼接正反向上LSTM的输出作为query的向量表示，使用`u`表示；document中`t`位置的token输出表示为正反向上LSTM的输出的拼接，使用`y(t)`表示，document的表示则是用document中所有token的加权平均来表示；权重矩阵`W`为网络在回答问题时对文档中特定位置的token的重视程度，即注意力；变量`s(t)`是token的规范化注意力表示；`r`为文档表示；然后使用query和document的表示作为分类模型的输入。

**1.1.3 Impatient Reader**

不难看出，**Attentive Reader将query作为一个整体来分析document中不同token的注意力**，但query中不同token本身的重要性也是不一样的，**Impatient Reader进一步分析query中每个token，尝试找到query中的token与document中哪几个token关联最大**，并且对于query中每个token都需要考虑到上一个token在document中积累的信息。图7展示了Impatient Reader中query和document的编码过程。

<img title="CNN&amp;Dailymail：TeachingMachinestoReadandComprehend_" 图片1="" src="https://img.mukewang.com/5ad96ef60001d06d07120434.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图7 Impatient Reader中query和document的编码过程</p>

Impatient Reader模型较Attentive Reader更为复杂，但在某些情况下，Impatient Reader效果可能并不是很好，因为在每读取query中的一个token就要通读原文一次，且还需要考虑上一个token在原文中相关的token，这样效率可能不高，且可能存在梯度弥散的问题。

**2 实验**
----

**2.0 基线方法**

（1）**多数基线(Maximum frequency)**选择上下文文档中最常见的实体
（2）**排除式多数(Exclusive frequency)**选择在上下文中最常见但在问题中没有出现的实体，即排除在问题中出现的实体。

**2.1 模型超参数设置**

所有深度学习模型均使用RmsProp优化方法，decay为0.95。

**（1）Deep LSTM Reader**

 - 隐藏层大小：[64, 128, 256]
 - LSTM层数：[1, 2, 4]
 - 学习率初始化：[1E-3, 5E-4, 1E-4, 5E-5]
 - batch size：[16, 32]
 - dropout：[0.0, 0.1, 0.2]

**（2）Attentive Reader & Impatient Reader**

 - 隐藏层大小：[64, 128, 256]
 - LSTM层数：1
 - 学习率初始化：[1E-4, 5E-5, 2.5E-5, 1E-5]
 - batch size：[8, 16, 32]
 - dropout：[0.0, 0.1, 0.2, 0.5]

**2.2 实验结果**

图8展示的是基线方法以及深度学习模型的实验结果对比。

<img title="CNN&amp;Dailymail：TeachingMachinestoReadandComprehend_" 图片1="" src="https://img.mukewang.com/5ad96f0f0001498207500484.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图8 基线方法以及深度学习模型的实验结果对比</p>

由图8可知，不管在CNN还是Daily Mail中，**深度学习模型普遍比基线方法和传统模型要好**，这也展现了深度学习在机器阅读理解的适用性；Uniform Reader没有使用注意力机制，其性能表现说明注意力机制在机器阅读理解中有效性，图9展示了其与Impatient Reader、Attentive Reader模型的P-R图；在CNN中Impatient Reader模型较Attentive Reader表现更好，但相差不大，而在Daily Mail中Attentive Reader模型较Impatient Reader表现更好。

<img title="CNN&amp;Dailymail：TeachingMachinestoReadandComprehend_" 图片1="" src="https://img.mukewang.com/5ad96f1d00018bf306940578.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图9 三种深度学习模型的P-R图（其中Uniform没有使用注意力机制）</p>