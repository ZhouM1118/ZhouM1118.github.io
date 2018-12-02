---
layout: post
title:  "spark调优基础-spark ui初探"
date:   2018-10-14
desc: "spark调优基础-spark ui初探"
keywords: "spark, 调优, ui"
categories: [Datamining]
tags: [spark, 调优, ui]
icon: icon-html
---

spark web ui getting started : https://mapr.com/blog/getting-started-spark-web-ui/

当一个Spark Application运行起来后，可以通过访问hostname:4040端口来访问UI界面。hostname是提交任务的Spark客户端ip地址，端口号由参数spark.ui.port(默认值4040，如果被占用则顺序往后探查)来确定。由于启动一个Application就会生成一个对应的UI界面，所以如果启动时默认的4040端口号被占用，则尝试4041端口，如果还是被占用则尝试4042，一直找到一个可用端口号为止。

<img src="https://img.mukewang.com/5bc2ffa800010c7f19960432.png" alt="图片描述" style="display:block; margin:auto; width:70%">

# **1 job页面**

这里包含了两部分：Event Timeline，事件发生的时间线信息；当前应用分析出来的所有任务，包括所有的excutors中action的执行时间等。这里会显示所有Active，Completed, Cancled以及Failed状态的Job。

## 1.1 Event Timeline

可以看到executor创建的时间点，以及某个action触发的算子任务，执行的时间，如下图所示：


<img src="https://img.mukewang.com/5bc2ffef000166f819070684.png" alt="图片描述" style="display:block; margin:auto; width:70%">


从上图可以看到，这里开了10个executor，可以通过“--num-executors 10”来设置。我们通过count() 的action来触发了一个job，可以在上图的右下角看到。通过这个时间图，可以快速的发现应用的执行瓶颈，触发了多少个action。

## 1.2 所有任务信息

在Event Timeline下面展示了所有的任务信息，这里的任务是触发了action的任务，transform和action的区别可以在[《spark必知必会的基本概念》](https://www.imooc.com/article/254085)中查看。任务包括已完成的、正在执行的、失败的。Description表示action的名字和所在的行号，这里的行号是精准匹配到代码的，所以通过它可以直接定位到任务所属的代码，这在调试分析的时候是非常有帮助的。Duration显示了该action的耗时，通过它也可以对代码进行专门的优化。最后的进度条，显示了该任务失败和成功的次数，如果有失败的就需要引起注意，因为这种情况在生产环境可能会更普遍更严重。

# 2 Stages页面

点击Description中的链接可以看到该action具体的stage页面，包括DAG图等详细信息。如下图所示：

<img src="https://img.mukewang.com/5bc3004d00015dae15350611.png" alt="图片描述" style="display:block; margin:auto; width:70%">


<img src="https://img.mukewang.com/5bc3006100010ae511490569.png" alt="图片描述" style="display:block; margin:auto; width:70%">


<img src="https://img.mukewang.com/5bc3007000013e2e19080192.png" alt="图片描述" style="display:block; margin:auto; width:70%">

在Spark中job是根据action操作来区分的，另外任务还有一个级别是stage，它是根据宽窄依赖来区分的。窄依赖是指前一个rdd计算能出一个唯一的rdd，比如map或者filter等；宽依赖则是指多个rdd生成一个或者多个rdd的操作，比如groupbykey reducebykey等，这种宽依赖通常会进行shuffle。因此Spark会根据宽窄依赖区分stage，某个stage作为专门的计算，计算完成后，会等待其他的executor，然后再统一进行计算。

中间就涉及到shuffle 过程，前一个stage 的 ShuffleMapTask 进行 shuffle write，把数据存储在 blockManager 上面， 并且把数据位置元信息上报到 driver 的 mapOutTrack 组件中，下一个 stage 根据数据位置元信息， 进行 shuffle read，拉取上个stage 的输出数据。

<img src="https://img.mukewang.com/5bc300a4000128b906400524.png" alt="图片描述" style="display:block; margin:auto; width:70%">

在里面可以看到应用的所有stage，stage是按照宽依赖来区分的，因此粒度上要比job更细一些。

**job stage task三个的关系**

 - job：一个 rdd 的 action 触发的动作，可以简单的理解为，当你需要执行一个 rdd 的 action 的时候，会生成一个job；
 - stage：一个 job 会被切分成 1 个或 1 个以上的 stage，然后各个 stage 会按照执行顺序依次执行。
 - task：stage 下的一个任务执行单元，一般来说，一个 rdd 有多少个 partition，就会有多少个 task，因为每一个task 只是处理一个 partition 上的数据。

**那么一个job的stage是如何划分的呢？**

stage的划分是以shuffle操作作为边界的。

从上图看出，这里产生了5个stage，各个 stage 会按照执行顺序依次执行。stage0读入了50.6G的数据，生成了1049.5M的数据并写入了磁盘中。

点击进去还可以看到详细的DAG图，鼠标移到上面，可以看到一些简要的信息。点击某一个stage，进入Stages页面，里面包含DAG图，并且看到executor和task的详细执行信息。DAG图也叫作血统图，标记了每个rdd从创建到应用的一个流程图，也是我们进行分析和调优很重要的内容。

<img src="https://img.mukewang.com/5bc301090001352511770703.png" alt="图片描述" style="display:block; margin:auto; width:70%">

<img src="https://img.mukewang.com/5bc301250001159d19000201.png" alt="图片描述" style="display:block; margin:auto; width:70%">

# 3 storage页面

我们所做的cache persist等操作，都会在这里看到，可以看出来应用目前使用了多少缓存，点击进去可以看到具体在每个机器上，使用的block的情况。

# 4 environment页面

里面展示了当前spark所依赖的环境，比如jdk,lib等等

# 5 executors页面

这里可以看到执行者申请使用的内存以及shuffle中input和output等数据。

这个页面比较常用，一方面通过它可以看出来每个excutor是否发生了数据倾斜(某一个partition上的数据显著多余其它partition，从而使得该部分的处理速度成为整个数据集处理的瓶颈。)

**数据倾斜是如何造成的：**

在Spark中，同一个Stage的不同Partition可以并行处理，而具有依赖关系的不同Stage之间是串行处理的。假设某个Spark Job分为Stage 0和Stage 1两个Stage，且Stage 1依赖于Stage 0，那Stage 0完全处理结束之前不会处理Stage 1。而Stage 0可能包含N个Task，这N个Task可以并行进行。如果其中N-1个Task都在10秒内完成，而另外一个Task却耗时1分钟，那该Stage的总时间至少为1分钟。换句话说，一个Stage所耗费的时间，主要由最慢的那个Task决定。

由于同一个Stage内的所有Task执行相同的计算，在排除不同计算节点计算能力差异的前提下，不同Task之间耗时的差异主要由该Task所处理的数据量决定。
Stage的数据来源主要分为如下两类
* 从数据源直接读取。如读取HDFS，Kafka
* 读取上一个Stage的Shuffle数据

另一方面可以具体分析目前的应用是否产生了大量的shuffle，是否可以通过数据的本地性或者减小数据的传输来减少shuffle的数据量。