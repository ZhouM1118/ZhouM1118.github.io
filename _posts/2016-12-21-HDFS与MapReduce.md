---
layout: post
title:  "HDFS与MapReduce"
date:   2016-12-21
desc: "HDFS与MapReduce"
keywords: "HDFS, MapReduce"
categories: [Datamining]
tags: [HDFS, MapReduce]
icon: icon-html
---

# **HDFS (Hadoop Distributed File System)**

## **数据块**

HDFS将文件分成一个个块，除了最后一个块的大小不一样，其余所有的块的大小都是一样的（128MB）。

## **数据复制**

Q：“一次写入，多次读取”，数据写入后，是否在所有结点中的dataNode中复制了一遍？

A：对的，这个是可配置的，可复制一遍或多遍。

## **数据副本的存放策略**

既然数据块在HDFS中有多个副本备份，那么数据副本应该在HDFS中怎么存放呢？

数据副本的存放需要考虑到数据的可靠性、可用性和网络带宽的利用率。

一个简单且没有有优化的策略是，将数据副本块放在不同的机架（一个机架上有很多不同的节点即DataNode）上，数据副本均匀放在集群中，有助于负载均衡，但需要将一个数据副本看写入到不同的机架上就务必会增加写的代价。

HDFS默认的副本系数为3，Hadoop的3个数据副本的存放策略：

**第一块：**在本机器下的HDFS目录下存储一个数据块；

**第二块：**不同的机架上的某个DataNode上存储一个数据块；

**第三块：**在该机器的同一个机架下的某台机器（也就是节点DataNode）上存储最后一个数据块。

## **负载均衡 Balancer**

需要考虑到网络带宽使用率，机器磁盘利用率等。

负载均衡程序balancer作为一个独立的进程与NameNode进程分开执行

具体过程：

1. 负载均衡服务Rebalancing Server从NameNode中获取所有的DataNode情况（包括DataNode的磁盘使用情况等）
2. Rebalancing Server计算哪些机器需要将数据移动，哪些机器可以接收移动的数据，哪一台机器的数据块可以移动到另一台机器中去
3. Rebalancing Server执行移动数据块操作，移动成功后删除原来的数据块

## **HDFS的Master/Slave主从架构**

**NameNode—Master：**管理HDFS的命名空间和和文件数据块

**DataNode—Slave：**数据存储节点

**SecondaryNameNode—辅助的NameNode：**为了降低单个的NameNode的宕机对系统造成影响，SecondaryNameNode会备份NameNode的相关数据

# **MapReduce（Map映射+Reduce化简）**

## **MapReduce过程**

map()负责将任务分散成多个子任务（按照键值对的形式，比如讲一篇文章分成多个map，每个map的key是字符串的偏移量，value是文件中一句字符串），将这些键值对作为输入，通过map()函数的处理（将每个map的value中的字符串分割为一个个单词），产生一个个中间map结果（将一个个单词作为map的key，value初始化为1），并将这些键值对结果写入本地磁盘。

MapReduce框架会自动将这些中间数据按照key值进行聚集，将key值相同的数据统一交给一个reduce()函数处理。

reduce()函数负责将分解后的多个任务的处理中间结果进行汇总（把key值相同的value值相加，得到一篇文章中不同单词出现的次数），产生最终结果的键值对（key为单词，value为单词出现的总次数）写入HDFS中。

## **MapReduce的Master/Slave主从架构**

**ResourcesManager—Master：**控制整个集群的计算资源分配

**NodeManager—Slave：**实际的计算节点