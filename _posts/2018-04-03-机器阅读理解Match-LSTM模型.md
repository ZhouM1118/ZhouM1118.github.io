---
layout: post
title:  "机器阅读理解Match-LSTM模型"
date:   2018-04-03
desc: "机器阅读理解Match-LSTM模型"
keywords: "机器阅读理解数据集, Match-LSTM"
categories: [Deeplearning]
tags: [机器阅读理解数据集, Match-LSTM]
icon: icon-html
---

**0 基于SQuAD数据集的通用模型架构**
-------------------

由于 SQuAD 的答案限定于来自原文，模型只需要判断原文中哪些词是答案即可，因此是**一种抽取式的 QA 任务而不是生成式任务**。几乎所有做 SQuAD 的模型都可以概括为同一种框架：**Embed 层，Encode 层，Interaction 层和 Answer 层**。Embed 层负责将原文和问题中的 tokens 映射为向量表示；Encode 层主要使用 RNN 来对原文和问题进行编码，这样编码后每个 token 的向量表示就蕴含了上下文的语义信息；Interaction 层是大多数研究工作聚焦的重点，该层主要负责捕捉问题和原文之间的交互关系，并输出编码了问题语义信息的原文表示，即 query-aware 的原文表示；最后 Answer 层则基于 query-aware 的原文表示来预测答案范围。图1展示了一个高层的神经 QA 系统基本框架。

<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac3707a0001da6806000337.jpg" alt="图片描述" style="display:block; margin:auto; width:70%">

<p style="text-align:center">图1 一个高层的神经 QA 系统基本框架</p>

**1 Match-LSTM模型架构**
----------------

图2展示了Match-LSTM模型架构图。

<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac370b700015cfa16040904.png" alt="图片描述" style="display:block; margin:auto; width:80%">

<p style="text-align:center">图2 Match-LSTM模型架构图</p>

在模型实现上，Match-LSTM 的主要步骤如下：

 1. Embed 层使用**词向量表示原文和问题**；
 2. Encode 层使用单向 LSTM 编码原文和问题 embedding；
 3. Interaction层对原文中每个词，**计算其关于问题的注意力分布**，并使用该注意力分布汇总问题表示，将原文该词表示和对应问题表示输入另一个 LSTM编码，得到**该词的 query-aware 表示**；
 4. 在反方向重复步骤 2，获得双向 query-aware 表示；
 5. Answer 层基于双向 query-aware 表示**使用 Sequence Model 或 Boundary Model预测答案范围**。

Match-LSTM模型的输入由两部分组成：段落（passage）和问题（question）。passage使用一个矩阵`P[d * P]`表示，其中`d`表示词向量维度大小，`P`表示passage中tokens的个数；question使用矩阵`Q[d * Q]`表示，其中`Q`表示question中tokens的个数。

Match-LSTM模型的输出即问题的答案有两种表示方法，其一是使用一系列整数数组`a = (a1, a2,…)`，其中`ai`是`[1, P]`中的一个整数，表示在段落中某个token具体的位置，这里的整数数组不一定是连续的，对应Sequence Model ；第二种表示方法是假设答案是段落中一段连续的token组合，即仅使用两个整数来表示答案`a = (as, ae)`，`as`表示答案在段落中开始的位置，`ae`则表示结束位置，`as`和`ae`是`[1, P]`中的整数，对应Boundary Model 。

故对Match-LSTM模型的训练集样本来说，其可用下面的**三维数组**来表示：

<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac3726700010b6b03200052.png" alt="图片描述" style="display:block; margin:auto; width:30%">

从图2可知，一个基本的Match-LSTM模型应该包含以下三层结构：

 1. **LSTM preprocessing Layer**：这层对passage和question进行预处理，得到其向量表示；
 2. **Match-LSTM Layer**：这层试图在passage中匹配question；
 3. **Answer Pointer(Ans-Ptr) Layer**：使用Ptr-Net从passage中选取tokens作为question的answer， Sequence Model 和 Boundary Model的主要区别就在这一层。

**LSTM preprocessing Layer**

LSTM preprocessing Layer的目的是将token的上下文信息包含到passage和question中的每个token的向量表示中。分别将passage和question输入LSTM preprocessing Layer，经过LSTM preprocessing Layer后，passage和question表示如下矩阵：

<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac372c600016a5706180078.png" alt="图片描述" style="display:block; margin:auto; width:60%">

`Hp`是passage的向量矩阵表示，其大小为`[l * P]`，l表示隐藏层的节点个数；`Hq`是question的向量矩阵表示，其大小为`[l * Q]`。

**Match-LSTM Layer**

将LSTM preprocessing Layer的输出作为这一层的输入，计算公式如下：

<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac372de0001273c09740152.png" alt="图片描述" style="display:block; margin:auto; width:70%">

其中，<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac373010001608707580052.png" alt="图片描述" style="width:60%">，这些参数都是模型需要训练的参数，<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac3733300010dc602000066.png" alt="图片描述" style="width:15%">是中间结果，<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac3734e0001d8fe02420066.png" alt="图片描述" style="width:15%">表示第`i-1`个token的隐藏层输出，<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac373640001718c00900052.png" alt="图片描述" style="width:7%">表示在列方向拓展Q列。

**对于`Gi`公式的维度变化：**

> [l * Q] = [l * l] * [l * Q] + ([l * l] * [l * 1] + [l * l] * [l * 1] + [l * 1]) * |Q|；

**对于`ai`公式的维度变化：**

> [1 * Q] = ([1 * l] * [l * Q] + 1 * |Q|)。

`ai`表示在passage的第`i`个token对question的注意力权重向量。

将注意力权重向量与question向量的乘积和passage中第`i`个token的隐藏层输出向量拼接成一维向量`Zi`，公式如下：

<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac3738c000116a502740144.png" alt="图片描述" style="display:block; margin:auto; width:25%">

其中，向量维度分别为：<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac373a30001165007100058.png" alt="图片描述" style="width:50%">

将向量Zi作为一个LSTM层的输入，得到

<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac373b60001352304780084.png" alt="图片描述" style="display:block; margin:auto; width:50%">

其中，隐藏层向量维度为：<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac373ce000105a201900074.png" alt="图片描述" style="width:15%">

将`P`个该隐藏层向量拼接成<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac373e10001516e02300058.png" alt="图片描述" style="width:15%">，为了提取上下文信息，反方向构建<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac373fa0001712a02420078.png" alt="图片描述" style="width:15%">，将两者拼接得到Match-LSTM Layer的输出：<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac37412000137bd02980154.png" alt="图片描述" style="display:block; margin:auto; width:20%">

**Answer Pointer Layer**

将Match-LSTM Layer的输出`Hr`作为这一层的输入。

Match-LSTM 的 Answer Pointer Layer 层包含了两种预测答案的模式，分别为 Sequence Model 和 Boundary Model。Sequence model提供的是一系列答案序列，但这些答案系列不一定在原文中是连续的；Boundary model是提供答案的开始与截止位置，在这两个位置中间所有的字就都是答案。Sequence Model 将答案看做是一个整数组成的序列，每个整数表示选中的 token 在原文中的位置，因此模型按顺序产生一系列条件概率，每个条件概率表示基于上轮预测的 token 产生的下个 token 的位置概率，最后答案总概率等于所有条件概率的乘积。Boundary Model 简化了整个预测答案的过程，只预测答案开始和答案结束位置，相比于 Sequence Model 极大地缩小了搜索答案的空间，最后的实验也显示简化的 Boundary Model 相比于复杂的 Sequence Model 效果更好，因此 Boundary Model 也成为后来的模型用来预测答案范围的标配。

Sequence Model的答案定位计算公式如下：

<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac374290001e74407560110.png" alt="图片描述" style="display:block; margin:auto; width:60%">

`k`是`[1, P+1]`中某个值，当`k=P+1`时，表示答案迭代计算答案位置概率结束。

Boundary Model的答案定位计算公式如下：

<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac3744000019f4506120110.png" alt="图片描述" style="display:block; margin:auto; width:60%">

**2 实验**
----

**模型参数设置如下：**

 - 隐藏层大小：`150`
 - 优化方法：`Adamax(β1 = 0.9，β2 = 0.999)`
 - minibatch：`30`
 - 没有用`L2`正则

实验结果对比如图3所示：

<img title="机器阅读理解Match-LSTM模型_" 图片1="" src="https://img.mukewang.com/5ac37472000179ad15620702.png" alt="图片描述" style="display:block; margin:auto; width:80%">

<p style="text-align:center">图3 各模型实验对比结果</p>

**3 参考文献**
------

\[1] 目前（2017年）机器阅读技术发展得如何？能达到什么水平？有哪些应用？ [2017-05-22]. https://www.zhihu.com/question/59280791/answer/172363224

\[2] Wang S, Jiang J. Machine Comprehension Using Match-LSTM and Answer Pointer[J]. 2016.