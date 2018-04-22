---
layout: post
title:  "SQuAD：机器阅读理解领域中的ImageNet"
date:   2018-04-22
desc: "SQuAD：机器阅读理解领域中的ImageNet"
keywords: "机器阅读理解数据集, SQuAD, ImageNet"
categories: [Deeplearning]
tags: [机器阅读理解数据集, SQuAD, ImageNet]
icon: icon-html
---

**0 介绍**
--------

上篇博文我们通过介绍机器阅读理解的奠基之作《Teaching Machines to Read and Comprehend》对CNN&DailyMail语料库进行了分析，但**由于CNN&DailyMail的answer只能是实体对象，且只能出现在原文中，而使得基于该语料库训练的模型缺乏推理能力**，于是斯坦福大学的Rajpurkar等人于2016年在自然语言处理的顶级会议EMNLP上发布了SQuAD语料库。[论文下载](https://arxiv.org/abs/1606.05250)。[数据集下载](https://rajpurkar.github.io/SQuAD-explorer/)。SQuAD通过众包的方式，**从wikipedia上的536篇文章提取超过10w个问题-答案对，且其中的答案是原文的一个片段而不是单一的实体对象**。图1展示了SQuAD语料库中的一个原文-问题-答案样本。

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc667600010bbc08260836.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图 1 SQuAD语料库中的一个原文-问题-答案样本</p>

**1 语料库对比**
-----------

**机器阅读理解是一个数据驱动的研究领域**，近几年不同的机构都发布了大大小小的机器阅读理解语料库，图2展示了机器阅读理解领域知名语料库对比。

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc66910001c26608220758.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图 2 机器阅读理解领域知名语料库对比</p>

其中，MCTest：660篇文章，每篇文章包含4个问题，每个问题有四个选项；WikiQA是定位答案在原文所在句子的位置，而SQuAD定位的是答案在原文哪个句子中哪些范围的单词；CNN&DailyMail是完形填空式风格的语料库，其答案一定是实体对象，而SQuAD的答案不一定是实体对象。

**2 SQuAD分析**
-------------

**2.1 多样的答案类型**

图3展示了SQuAD语料库中答案的多种类型。

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc66ab00013e6a08180492.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图 3 SQuAD语料库中答案的多种类型</p>

从图3中可看出，19.8%的答案为日期和其他数字类；有32.6%的答案为三种不同类型的专有名词：人名、地名以及其它实体对象；31.8%为名词短语；其余15.8%由形容词短语、动词短语、从句和其他类型组成。

**2.2 SQuAD语料库的推理能力**

**Rajpurkar通过构造问题与答案所在原文的句子之间的多种差异类型来增加SQuAD语料库在问题求解时的推理能力。**图4展示了SQuAD语料库中问题与答案所在原文的句子之间的差异类型。

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc66c20001695e16961042.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图 4 SQuAD语料库中问题与答案所在原文的句子之间的差异类型</p>

由图4可看出，

 - Lexical variation-词汇差异：
    同义词推理，如called-referred
    背景知识积累/推理，如governing bodies-The European Parliament and the Council of the European Union

 - Syntactic variation-句法差异：
即使将问题转化为陈述句后，其与答案所在原文句子的句法结构具有差异

 - Multiple sentences reasoning-答案需要在原文多个句子中进行推理才能得到

 - Ambiguous-众包问题无法有唯一的答案，或众包得到的答案并不对

**2.3 问题和答案所在句子的句法差异性**

图5 展示了问题和答案所在句子的句法差异性计算过程

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc670f0001dbd908160364.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图 5 问题和答案所在句子的句法差异性计算过程</p>

首先**找出两个句子（问题和答案所在原文句子）的相似词**，比如first，然后再question中找到wh开头的词（即what、why、which等等），而答案所在原文句子则以句子的第一个词作为开头，**计算两个句子的编辑距离**；对于两个句子有多个相似词，则**选最小编辑距离作为句法差异度的度量值**。两个句子的句法差异度的度量范围在0~8之间，如图6所示，编辑距离小不代表问题很容易回答，因为问题在原文中可能有多个同样编辑距离小的句子。

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc67290001db2207900308.png" alt="图片描述" style="display:block; margin:auto; width:60%">

<p style="text-align:center">图6 (a) 编辑距离为0的两个句子</p>

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc673a0001094d16440252.png" alt="图片描述" style="display:block; margin:auto; width:85%">

<p style="text-align:center">图6 (b) 编辑距离为6的两个句子</p>

图7 展示了SQuAD语料库在句法差异的分布情况。

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc674d0001d3f307660416.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图 7 SQuAD语料库在句法差异的分布情况</p>

**3 模型及实验结果**
-------------

Rajpurkar使用逻辑回归算法构建机器阅读理解模型，分析开发集发现，有77.3%的答案来源于原文，那么在训练过程中，规定答案一定是来源于原文，如果答案不出现在原文，则将原文中能表达答案同等意思的最小部分作为答案。

**3.1 特征工程**

模型的特征工程总共包含了**1.8亿个特征**，其中大部分是词汇化特征或依赖树路径特征，图8 展示了特征工程中特征详细信息。

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc676c00013e7917001174.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图8 特征工程中特征详细信息</p>

由图8可以看出，

 1. **匹配词(matching word)、双字词的词频(bigram frequencies)和词根匹配(root match)特征**帮助模型定位候选答案所在句子；
 2. **长度特征(Length features)**帮助模型选择具有共同长度和位置的答案跨度；
 3. **跨度中的词频(span word frequencies)**特征帮助模型剔除偏僻词；
 4. **成分类型特征(Constituent label)和跨度中单词的词性标注特征(span POS tag features)**帮助模型确定正确答案类型；
 5. **词法特征(lexicalized features)和依赖树路径特征(dependency tree path features)**来定量问题与候选答案句子之间的词法和句法多样性。

**3.2 参数设置**

 1. 损失函数：交叉熵损失
 2. 优化方法：AdaGrad
 3. 初始化学习率：0.1
 4. 正则化：L2(0.1/batch size)

**3.3 基线方法**

1. **sliding-window**
对于每一个候选答案所在的句子，剔除候选答案本身后，计算其与问题之间单个单词和两个单词的重叠个数，重叠个数最大的句子即为正确答案。

2. **基于距离(distance-based )的计算**
计算问题与候选答案所在句子之间的距离，距离越小的句子即为正确答案。

**3.4 评价方法**

1. 完全匹配（Exact Match），即预测的答案与正确答案完全相符
2. F1值

**3.5 实验结果**

图9 展示了不同模型实验结果，我们可以看到**逻辑回归模型远优于基线方法**，但离人类的表现还差很远，说明基于SQuAD语料库的机器阅读理解研究还有很大空间。

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc683a0001518508100366.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图9 不同模型实验结果对比</p>

图 10 展示了逻辑回归模型在剔除某种特征后模型的实验结果。我们可以看出**词法特征(lexicalized features)和依赖树路径特征(dependency tree path features)对模型影响最大**，而剔除其它特征几乎不会对逻辑回归模型造成很大的影响。

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc68520001a1f505820592.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图 10 逻辑回归模型在剔除某种特征后的实验结果</p>

图 11 展示了模型在不同答案类型的表现情况。我们可以看到**模型对数字类和实体类表现良好，而人对所有类型答案的表现都差不多**。

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc685f0001b9d207640536.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图11 模型在不同答案类型的表现情况</p>

图 12 展示了在不同问题与答案所在句子之间的句法差异对模型的影响。我们可以看出**随着句法差异越来越大，模型性能逐渐降低，但人类的表现却很稳定。**

<img title="SQuAD：机器阅读理解领域中的&quot;ImageNet&quot;_" 图片1="" src="https://img.mukewang.com/5adc68a60001509908240520.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图 12 不同问题与答案所在句子之间的句法差异对模型的影响</p>