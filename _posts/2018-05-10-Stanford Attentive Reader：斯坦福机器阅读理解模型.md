---
layout: post
title:  "Stanford Attentive Reader：斯坦福机器阅读理解模型"
date:   2018-05-10
desc: "Stanford Attentive Reader：斯坦福机器阅读理解模型"
keywords: "Stanford Attentive Reader, 机器阅读理解, 深度学习"
categories: [Deeplearning]
tags: [Stanford Attentive Reader, 机器阅读理解, 深度学习]
icon: icon-html
---

# **0 前言**

Stanford Attentive Reader是斯坦福在2016年的ACL会议上的《A Thorough Examination of the CNN/Daily Mail Reading Comprehension Task》（[论文地址](http://arxiv.org/abs/1606.02858)，[GitHub源码](https://github.com/danqi/rc-cnn-dailymail)，代码要求`Python 2.7，Theano >= 0.7，Lasagne 0.2.dev1`，代码整体风格非常简洁易懂）发布的一个机器阅读理解模型，**这篇论文通过设计一个基于双线性匹配函数的深度学习模型，完成在CNN&Daily Mail完形填空式数据集上的机器阅读理解任务**，Stanford Attentive Reader的性能要比上篇博文[《ASReader：一个经典的机器阅读理解深度学习模型》](https://zhoum1118.github.io/deeplearning/2018/05/09/ASReader-%E4%B8%80%E4%B8%AA%E7%BB%8F%E5%85%B8%E7%9A%84%E6%9C%BA%E5%99%A8%E9%98%85%E8%AF%BB%E7%90%86%E8%A7%A3%E6%B7%B1%E5%BA%A6%E5%AD%A6%E4%B9%A0%E6%A8%A1%E5%9E%8B.html)中ASReader模型和[《CNN&Dailymail：Teaching Machines to Read and Comprehend》](https://zhoum1118.github.io/deeplearning/2018/04/15/CNN&Dailymail-Teaching-Machines-to-Read-and-Comprehend.html)的Attentive Reader模型性能更好。

这篇论文不仅探索了深度学习神经网络在机器阅读理解的应用，还构建了一个基于boosted决策树森林的机器阅读理解模型，下面我们就这两种方法进行分析，同时给出模型基本实验设置以及实验结果，实验数据集是CNN&Dailymail，在[《CNN&Dailymail：Teaching Machines to Read and Comprehend》](https://zhoum1118.github.io/deeplearning/2018/04/15/CNN&Dailymail-Teaching-Machines-to-Read-and-Comprehend.html)有详细介绍，这里就不再赘述了。

# **1 基于boosted决策树森林的机器阅读理解模型**

由于数据集是完形填空式，且答案一定是实体类单词，所以特征一般围绕文档中实体来提取的。通过特征工程构建每个实体类单词`e`的特征向量`f_p,q(e)`，其中`q`表示文章（passage），`q`代表问题（question），特征工程中包含的特征有：

 - 实体e是否在文章中出现；
 - 实体e是否在问题中出现；
 - 实体e在文章中的词频；
 - 实体e在文章中第一次出现的位置；
 - 实体e的n-gram匹配特征：计算在文章中实体e周围的的文本（左右1到2个单词）与问题中答案占位符周围的文本间的n-gram匹配度；
 - 词距离特征：计算实体与问题中每个关键词的最小词距离的平均值；
 - 句共现特征：在问题中的实体e与另一个实体或动词一起出现是否在文章的某些句子中也是一起出现；
 - 依存句法特征：在[《一种基于模糊依存关系匹配的问答模型构建方法》](https://zhoum1118.github.io/deeplearning/2018/04/30/%E4%B8%80%E7%A7%8D%E5%9F%BA%E4%BA%8E%E6%A8%A1%E7%B3%8A%E4%BE%9D%E5%AD%98%E5%85%B3%E7%B3%BB%E5%8C%B9%E9%85%8D%E7%9A%84%E9%97%AE%E7%AD%94%E6%A8%A1%E5%9E%8B%E6%9E%84%E5%BB%BA%E6%96%B9%E6%B3%95.html)有详细介绍。

将机器阅读理解看成是一个排序问题，并使用`RankLib`包的`LambdaMART`来构建**boosted决策树森林模型**。

# **2 基于深度学习的机器阅读理解模型：Stanford Attentive Reader**

图1展示了Stanford Attentive Reader模型结构图。

<img title="StanfordAttentiveReader：斯坦福机器阅读理解模型_" 图片1="" src="https://img.mukewang.com/5af3eb2400015bd813980728.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图1 Stanford Attentive Reader模型结构图</p>

## **2.1 Encoding层**

首先基于所有的文章和问题中构建一个词向量矩阵`E`，`E`的维度为`d * |V|`，`d`代表词向量的维度，`|V|`表示词字典大小；然后对文章（passage）和问题（question）进行编码（encode）。**对于passage的编码**，使用了一个单层双向GRU，对于文档中每个词用GRU正向隐藏层的输出与GRU反向隐藏层的输出的拼接向量来表示，这时候的拼接向量包含了这个词的上下文信息，过程如下图所示：

<img title="StanfordAttentiveReader：斯坦福机器阅读理解模型_" 图片1="" src="https://img.mukewang.com/5af3eb510001e5be07340164.png" alt="图片描述" style="display:block; margin:auto; width:40%">

**passage编码后可表示为**：<img title="StanfordAttentiveReader：斯坦福机器阅读理解模型_" 图片1="" src="https://img.mukewang.com/5af3eb600001563604000076.png" alt="图片描述" style="width:24%">

**对于question的编码**，由于问题一般比文档要短很多，所以这时候将问题看作一个整体，同样使用了一个单层双向GRU来encode，但这时候不考虑每个词的隐藏层的输出，而是只考虑正向GRU最后的输出与反向GRU最后的输出，将这两个输出拼接作为问题的encode向量，用`q`来表示。

## **2.2 Attention层**

这一层的目标是计算文章中每个实体词的embedding表示`p_i`与问题embedding表示`q`的相关度，并选择与问题最相关的实体词作为问题答案。**不同ASReader模型的求点积，Stanford Attentive Reader使用了双线性函数作为匹配函数**，然后累加相同词在不同文章不同位置的相似度。双线性函数可以计算`q`和`p_i`之间的相似性，比用点积更灵活。计算公式如下：

<img title="StanfordAttentiveReader：斯坦福机器阅读理解模型_" 图片1="" src="https://img.mukewang.com/5af3ebb30001536005260144.png" alt="图片描述" style="display:block; margin:auto; width:30%">

其中，`W_s`即为双线性项，其维度为`h * h`，`h`为单层双向GRU隐藏层大小，`O`为这一层的输出向量，代表了文章中所有单词的带attention的embedding表示。

**双线性函数的核心代码**：

```
import theano.tensor as T
def get_output_for(self, inputs, **kwargs):
        # inputs[0]: batch * len * h
        # inputs[1]: batch * h
        # W: h * h
        M = T.dot(inputs[1], self.W).dimshuffle(0, 'x', 1)
        alpha = T.nnet.softmax(T.sum(inputs[0] * M, axis=2))
        if len(inputs) == 3:
            alpha = alpha * inputs[2]
            alpha = alpha / alpha.sum(axis=1).reshape((alpha.shape[0], 1))
        return T.sum(inputs[0] * alpha.dimshuffle(0, 1, 'x'), axis=1)
```

## **2.3 Prediction层**

不同于原有模型用一层非线性层对`O`和question embedding进行结合再做预测，而是直接在`O`上加一层softmax层计算概率最大的答案，计算公式如下：

<img title="StanfordAttentiveReader：斯坦福机器阅读理解模型_" 图片1="" src="https://img.mukewang.com/5af3ec4800010cd504760100.png" alt="图片描述" style="display:block; margin:auto; width:30%">

**从同一年的ACL会议两篇论文分析发现，Stanford Attentive Reader模型与ASReader模型步骤基本一致，只是在Attention层中，匹配函数有所不同，说明在CNN&Dailymail数据集上的机器阅读理解模型在这个时候模型基本无太大差异，重要的研究点在于匹配函数。**

# **3 实验设置及结果**

## **3.1 实验设置**

 - 优化函数：SGD
 - 词向量维度：100（使用预训练好的100维glove词向量）
 - 学习率：0.1
 - 损失函数：-logP(a|q, d)
 - GRU网络中的权值初始化：满足高斯分布N(0, 0.1)
 - 隐藏层大小h：CNN(128)，Dailymail(256)
 - Attention层权重矩阵初始化范围：[-0.01, 0.01]
 - batch size：32
 - dropout：0.2

## **3.2 实验结果**

<img title="StanfordAttentiveReader：斯坦福机器阅读理解模型_" 图片1="" src="https://img.mukewang.com/5af3ec8a0001cd2316700974.png" alt="图片描述" style="display:block; margin:auto; width:50%">

同时对于基于boosted决策树森林的机器阅读理解模型，实验分析了不同特征对模型的影响。图2展示了在CNN数据集中模型的特征消融分析实验结果，Accuracy表示从特征工程中排除相应特征后的准确性，**准确率越低表示对应特征越重要，从图2可以看出，词频特征和n-gram特征最重要。**

<img title="StanfordAttentiveReader：斯坦福机器阅读理解模型_" 图片1="" src="https://img.mukewang.com/5af3ec9800016eb808200620.png" alt="图片描述" style="display:block; margin:auto; width:50%">

<p style="text-align:center">图2 CNN数据集中模型的特征消融分析实验结果</p>