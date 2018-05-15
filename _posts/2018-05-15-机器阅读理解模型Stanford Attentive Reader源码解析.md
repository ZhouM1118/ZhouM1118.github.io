---
layout: post
title:  "机器阅读理解模型Stanford Attentive Reader源码解析"
date:   2018-05-15
desc: "机器阅读理解模型Stanford Attentive Reader源码解析"
keywords: "Stanford Attentive Reader, 机器阅读理解, 源码解析"
categories: [Deeplearning]
tags: [Stanford Attentive Reader, 机器阅读理解, 源码解析]
icon: icon-html
---

# **0 前言**

Stanford Attentive Reader是斯坦福在2016年的ACL会议上的《A Thorough Examination of the CNN/Daily Mail Reading Comprehension Task》（[论文地址](http://arxiv.org/abs/1606.02858)，[GitHub源码](https://github.com/danqi/rc-cnn-dailymail)，代码要求Python 2.7，Theano >= 0.7，深度学习框架[Lasagne 0.2.dev1](http://lasagne.readthedocs.io/en/latest/index.html)，代码整体风格非常简洁易懂）发布的一个机器阅读理解模型。我们对Stanford Attentive Reader模型的源码进行了解析，力求通过分析其源码来深入研究一个机器阅读理解模型是如何工作的。

采用的数据集有两个：CNN和Daily Mail，下载地址分别为[CNN](http://cs.stanford.edu/~danqi/data/cnn.tar.gz) (546M)和[Daily Mail](http://cs.stanford.edu/~danqi/data/dailymail.tar.gz) (1.4G)；100维glove词向量[下载地址](http://nlp.stanford.edu/data/glove.6B.zip)。

# **1 载入数据load_data**

**这部分主要是载入训练集、验证集以及测试集的CNN和Daily Mail数据**，输入参数主要有三个：

 1. in_file：表示数据文件地址；
 2. max_example：如果在debug模式下，该值为100，表示只采用数据集中前100个样本，如果在非debug模式下，该值为None，表示采用数据集中全部数据；
 3. relabeling：bool型，如果要求对数据集中实体类单词重新编号，则该值为True，若要求重新编号则按照实体类单词出现的顺序进行编号，默认为True。

**输出为文档集合、问题集合和答案集合。**

从load_data输入输出可看出，其功能主要是分别提取每个数据集样本中的文档、问题以及答案，并按对应顺序放到三个不同的list中，从`answer = entity_dict[answer]`中可以看出，**答案一定是文档或问题中的某个实体类单词。**

**源码为：**

```
def load_data(in_file, max_example=None, relabeling=True):
    documents = []
    questions = []
    answers = []
    num_examples = 0
    f = open(in_file, 'r')
    while True:
        line = f.readline()
        if not line:
            break
        question = line.strip().lower()
        answer = f.readline().strip()
        document = f.readline().strip().lower()

        if relabeling:
            q_words = question.split(' ')
            d_words = document.split(' ')
            assert answer in d_words

            entity_dict = {}
            entity_id = 0
            for word in d_words + q_words:
                if (word.startswith('@entity')) and (word not in entity_dict):
                    entity_dict[word] = '@entity' + str(entity_id)
                    entity_id += 1

            q_words = [entity_dict[w] if w in entity_dict else w for w in q_words]
            d_words = [entity_dict[w] if w in entity_dict else w for w in d_words]
            answer = entity_dict[answer]

            question = ' '.join(q_words)
            document = ' '.join(d_words)

        questions.append(question)
        answers.append(answer)
        documents.append(document)
        num_examples += 1

        f.readline()
        if (max_example is not None) and (num_examples >= max_example):
            break
    f.close()
    return (documents, questions, answers)
```

载入训练集和验证集数据后，则开始构建单词字典。

# **2 单词字典build_dict**

输入为文档集合和问题集合`sentences`；最大词数`max_words`，默认为50000。

输出即为文档和问题集合的单词字典`word_dict`，`key`为单词，`value`为按照词频排序的序号。

build_dict使用了Python集合类的`Counter`子类来完成文档与问题的单词统计，`Counter`可以便捷和快速地进行哈希的对象计数，并返回一个无序的容器，元素被作为字典的`key`存储，它们的计数作为字典的`value`存储。然后使用`most_common`方法从多到少返回一个有前`max_words`多的元素的列表`ls`，相同数量的元素次序任意。最后返回一个由ls组成的单词字典，值得注意的是，`ls`列表的数据从字典的第三个开始填充，单词字典第一个为“无法识别的词”即字典第一个元素为`”<UNK>”:0`，第二个为分隔符即`”|||”:1`。

**源码为:**

```
def build_dict(sentences, max_words=50000):
    word_count = Counter()
    for sent in sentences:
        for w in sent.split(' '):
            word_count[w] += 1

    ls = word_count.most_common(max_words)

    return {w[0]: index + 2 for (index, w) in enumerate(ls)}
```

**把机器阅读理解问题看作一个分类问题**。

```
entity_markers = list(set([w for w in word_dict.keys() if w.startswith('@entity')] + train_examples[2]))
entity_markers = ['<unk_entity>'] + entity_markers
entity_dict = {w: index for (index, w) in enumerate(entity_markers)}
args.num_labels = len(entity_dict)
```

# **3 embedding字典**

根据单词字典`word_dict`来构建嵌入层`gen_embeddings`，`gen_embeddings`的输入包含四个参数：

 1. `word_dict`：表示单词字典；
 2. `dim`：表示词向量维度；
 3. `in_file`：如果为None，则表示不使用预训练好的词向量来初始化嵌入层，这时使用第四个参数的参数初始化方法`lasagne.init.Uniform()`，即对嵌入层权值进行均匀分布初始化，如果不为None，则应该输入预训练词向量的文件地址，这里默认使用的是100维的glove词向量来初始化嵌入层，值得注意的是，类似于“@entity1”这种单词显然不包含在glove词向量中，那么对于这类词也采用均匀分布初始化的方法；
 4. `init`：默认为使用深度学习框架`lasagne`的均匀分布初始化方法。

`gen_embeddings`的输出即为`embeddings`字典，`key`表示`word_dict`中对应词的`value`，即按照词频排序的序号，`value`为一个列表，表示`key`的词向量，词向量中的值为`float`类型。

**源码为：**

```
def gen_embeddings(word_dict, dim, in_file=None,
                   init=lasagne.init.Uniform()):

    num_words = max(word_dict.values()) + 1
    embeddings = init((num_words, dim))

    if in_file is not None:
        pre_trained = 0
        for line in open(in_file).readlines():
            sp = line.split()
            if sp[0] in word_dict:
                pre_trained += 1
                embeddings[word_dict[sp[0]]] = [float(x) for x in sp[1:]]
    return embeddings
```

# **4 Encode层**

将构建的`embeddings`字典来初始化`lasagne.layers.EmbeddingLayer`的权重，这里默认使用双向GRU（`lasagne.layers.GRULayer`）来构建文档和问题的encode层，在`rnn_layer`中的参数`backwards`表示了是否使用双向GRU。对于文档的encode，即`name=‘d’`的`stack_rnn`，其中参数`only_return_final`表示是否要只返回GRU的最终输出，`args.att_func`默认为`bilinear`，所以`only_return_final`为`False`，即对于文档的encode是使用GRU的正向与反向的隐藏层输出来构建的；对于问题的encode，即`name=‘q’`的`stack_rnn`，`only_return_final=True`代表问题的encode是使用GRU的正向与反向的最终输出来构建的。

**源码为：**

```
l_in1 = lasagne.layers.InputLayer((None, None), in_x1)
l_mask1 = lasagne.layers.InputLayer((None, None), in_mask1)
l_emb1 = lasagne.layers.EmbeddingLayer(l_in1, args.vocab_size, args.embedding_size, W=embeddings)

l_in2 = lasagne.layers.InputLayer((None, None), in_x2)
l_mask2 = lasagne.layers.InputLayer((None, None), in_mask2)
l_emb2 = lasagne.layers.EmbeddingLayer(l_in2, args.vocab_size, args.embedding_size, W=l_emb1.W)

network1 = nn_layers.stack_rnn(l_emb1, l_mask1, args.num_layers, args.hidden_size,
                               grad_clipping=args.grad_clipping,
                               dropout_rate=args.dropout_rate,
                               only_return_final=(args.att_func == 'last'),
                               bidir=args.bidir,
                               name='d',
                               rnn_layer=args.rnn_layer)

network2 = nn_layers.stack_rnn(l_emb2, l_mask2, args.num_layers, args.hidden_size,
                               grad_clipping=args.grad_clipping,
                               dropout_rate=args.dropout_rate,
                               only_return_final=True,
                               bidir=args.bidir,
                               name='q',
                               rnn_layer=args.rnn_layer)
```

`stack_rnn`封装了`lasagne.layers.GRULayer`，对外使用`stack_rnn`来构建双向GRU。

```
def stack_rnn(l_emb, l_mask, num_layers, num_units,
              grad_clipping=10, dropout_rate=0.,
              bidir=True,
              only_return_final=False,
              name='',
              rnn_layer=lasagne.layers.LSTMLayer):

    def _rnn(backwards=True, name=''):
        network = l_emb
        for layer in range(num_layers):
            if dropout_rate > 0:
                network = lasagne.layers.DropoutLayer(network, p=dropout_rate)
            c_only_return_final = only_return_final and (layer == num_layers - 1)
            network = rnn_layer(network, num_units,
                                grad_clipping=grad_clipping,
                                mask_input=l_mask,
                                only_return_final=c_only_return_final,
                                backwards=backwards,
                                name=name + '_layer' + str(layer + 1))
        return network

    network = _rnn(True, name)
    if bidir:
        network = lasagne.layers.ConcatLayer([network, _rnn(False, name + '_back')], axis=-1)
    return network
```

# **5 Attention层**

从Encode层得到文档的上下文表示`network1`以及问题的上下文表示`network2`，使用一个双线性函数`nn_layers.BilinearAttentionLayer`作为`network1`和`network2`的匹配函数。

```
att = nn_layers.BilinearAttentionLayer([network1, network2], args.rnn_output_size, mask_input=l_mask1)
```

**核心计算代码为：**

```
M = T.dot(inputs[1], self.W).dimshuffle(0, 'x', 1)
alpha = T.nnet.softmax(T.sum(inputs[0] * M, axis=2))
if len(inputs) == 3:
    alpha = alpha * inputs[2]
    alpha = alpha / alpha.sum(axis=1).reshape((alpha.shape[0], 1))
return T.sum(inputs[0] * alpha.dimshuffle(0, 1, 'x'), axis=1)
```

# **6 全连接层Dense**

```
network = lasagne.layers.DenseLayer(att, args.num_labels, nonlinearity=lasagne.nonlinearities.softmax)
```

其中的`att`为双线性函数的输出，`args.num_labels`为数据集中各类`@entity`，可参考单词字典`build_dict`，使用非线性激活函数`softmax`，得到每个`@entity`的为答案的概率。

**模型使用SGD优化函数：** `updates = lasagne.updates.sgd(loss, params, args.learning_rate)`

**损失函数：**`lasagne.objectives.categorical_crossentropy(train_prediction, in_y)`，计算正确答案和预测答案的交叉熵。

# **7 可能遇到的问题：**

```
ImportError: cannot import name downsample
```

**解决方法：**

这主要是因为你安装的时候直接使用了`pip install lasagne`，这样安装的是`lasagneV0.1`，而这个模型需要的是`lasagneV0.2`，查看官网发现需使用`pip install --upgrade https://github.com/Lasagne/Lasagne/archive/master.zip来安装lasagneV0.2`