---
layout: post
title:  "性能对比：collections.sort vs treeSet sort vs java8 stream.sorted"
date:   2018-08-29
desc: "性能对比：collections.sort vs treeSet sort vs java8 stream.sorted"
keywords: "collections.sort, treeSet,stream.sorted"
categories: [Java]
tags: [collections.sort, treeSet,stream.sorted]
icon: icon-html
---

# **0 写在前面的话**

在项目中有一个排序问题，考虑到未来需要排序的数据量可能很大，想用一个性能较好的排序算法，现在有三套解决方法：jdk提供的集合的sort方法（Collections.sort）、一个可排序的数据结构TreeSet、Java8中流的排序（stream.sorted）。

我们都知道，TreeSet的底层是用红黑树实现的，它在调用集合上的add方法时，会始终保持集合中的数据排序，而Collection s.Sort()方法则会对整个集合数据进行排序，其底层有两种sort方法：legacyMergeSort（一种老的归并排序，默认是不使用，在未来版本会被移除）和TimSort.sort（这是一种归并排序merge sort和插入排序insertion sort的排序算法，如果待排序的数据大小小于32，直接使用二分插入排序），所以TreeSet的add()的时间复杂度为log(N)，Collection的时间复杂度为O(n*log(N)。但如果要排序相同大小的数据，那么TreeSet排序和Collections.sort的复杂性是相同的，因为使用TreeSet时会add n次。

**补充下 TimSort 的算法思想**：*为了减少对升序部分的回溯和对降序部分的性能倒退，将输入按其升序和降序特点进行了分区。排序的输入的单位不是一个个单独的数字，而是一个个的块-分区。其中每一个分区叫一个run。针对这些 run 序列，每次拿一个 run 出来按规则进行合并。每次合并会将两个 run合并成一个 run。合并的结果保存到栈中。合并直到消耗掉所有的 run，这时将栈上剩余的 run合并到只剩一个 run 为止。这时这个仅剩的 run 便是排好序的结果。*

# **1 排序方法性能对比**

本文主要是对三者排序方法的性能对比实验，所以不深究底层代码实现，直接上测试代码：

	import com.google.common.base.Ticker;
	import com.google.common.collect.Lists;
	import org.junit.Test;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;
	
	import java.util.Collections;
	import java.util.List;
	import java.util.TreeSet;
	import java.util.stream.Collectors;
	
	/**
	 * @author ming.zhou
	 * @date 2018/8/29
	 */
	public class TestSetSort {
	
	    private static final Logger LOGGER = LoggerFactory.getLogger(TestSetSort.class);
	
	    private static final int SIZE = 1000;
	
	    private List<Integer> init(){
	        List<Integer> datas = Lists.newArrayList();
	        for (int i = 0; i < SIZE; i ++){
	            datas.add(i);
	        }
	        Collections.shuffle(datas);
	        return datas;
	    }
	
	    @Test
	    public void testCollectionSort(){
	        List<Integer> datas = init();
	
	        long startTime = Ticker.systemTicker().read();
	
	        Collections.sort(datas);
	
	        LOGGER.info("sorted data by collections, result : {}", datas);
	
	        LOGGER.info("collection sort use time : {}", (Ticker.systemTicker().read() - startTime) / 1000);
	    }
	
	    @Test
	    public void testTreeSet(){
	        List<Integer> datas = init();
	
	        long startTime = Ticker.systemTicker().read();
	
	        TreeSet<Integer> treeSet = new TreeSet<>(datas);
	
	        LOGGER.info("sorted data by treeSet, result : {}", treeSet);
	
	        LOGGER.info("tree set sort use time : {}", (Ticker.systemTicker().read() - startTime) / 1000);
	    }
	
	    @Test
	    public void testJava8Sort(){
	        List<Integer> datas = init();
	
	        long startTime = Ticker.systemTicker().read();
	
	        List<Integer> result = datas.stream()
	                .sorted()
	                .collect(Collectors.toList());
	
	        LOGGER.info("sorted by java8 stream, result : {}", result);
	        LOGGER.info("java8 stream sort use time : {}", (Ticker.systemTicker().read() - startTime) / 1000);
	    }
	
	}

**实验结果：**

	size : 1000
	collection sort use time : 2257
	tree set sort use time : 29412
	java8 stream sort use time : 121645
	
	size : 10000
	collection sort use time : 39461
	tree set sort use time : 45831
	java8 stream sort use time : 113569
	
	size : 10000 * 10
	collection sort use time : 263307
	tree set sort use time : 202949
	java8 stream sort use time : 424212
	
	size : 10000 * 100
	collection sort use time : 2578863
	tree set sort use time : 2646424
	java8 stream sort use time : 924681

# **2 总结：**

我们可以看到Collections.sort比TreeSet的性能要更好些，分析后可能是由于TreeSet在红黑树中新插入节点也需要消耗一定的时间，且每次插入后需要对原有的集合进行修改以使得新集合底层仍然是红黑树。 而且TreeSet只能使用一个比较器，但你却可以为Collection提供不同的比较器。Java8的流排序的效率相比前两个就有点低了，但Java8的流排序可以省去构造冗长的比较器，编写更简洁的代码。