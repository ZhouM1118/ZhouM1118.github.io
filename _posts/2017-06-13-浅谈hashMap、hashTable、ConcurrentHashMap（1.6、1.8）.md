---
layout: post
title:  "浅谈hashMap、hashTable、ConcurrentHashMap（1.6、1.8）"
date:   2017-06-13
desc: "浅谈hashMap、hashTable、ConcurrentHashMap（1.6、1.8）"
keywords: "hashMap, hashTable, ConcurrentHashMap"
categories: [Java]
tags: [hashMap, hashTable, ConcurrentHashMap]
icon: icon-html
---

最近看Spring IoC源码的时候，发现Spring IoC中的BeanDefinition的注册采用ConcurrentHashMap<String, BeanDefinition>(256)来存储BeanDefinition，这让我想起了HashMap、HashTable以及ConcurrentHashMap的区别与联系，现在总结这三个类的实现原理和区别，在大脑里也重新再整理下这部分知识点。

**一、HashMap**

**1.1 HashMap的定义**

HashMap是最常用的集合类框架之一，它实现了Map接口，所以存储的元素也是键值对映射的结构，并允许使用null值和null键，其内元素是无序的，如果要保证有序，可以使用LinkedHashMap。

`public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable{}`

**1.2 hashMap的基本方法**

	//构造一个具有默认初始容量 (16) 和默认加载因子 (0.75) 的空 HashMap。
	HashMap()
	//构造一个带指定初始容量和默认加载因子 (0.75) 的空 HashMap。
	HashMap(int initialCapacity)
	//构造一个带指定初始容量和加载因子的空 HashMap。
	HashMap(int initialCapacity, float loadFactor)
	//构造一个映射关系与指定 Map 相同的 HashMap。
	HashMap(Map<? extendsK,? extendsV> m)

在hashMap的size达到initialCapacity*loadFactor时会对hashMap进行扩容，值得注意的是hashMap的size大小总是2的幂次方。初始容量和加载因子的选取也是影响HashMap性能的原因之一，加载因子过高虽然减少了空间开销，但同时也增加了查找某个条目的时间；加载因子过低也可能容易导致HashMap执行rehash操作。

在HashMap中我们直接接触的最常用的两个方法就是get和put方法

	public V put(K key, V value) {
	    return putVal(hash(key), key, value, false, true);
	}
	
	public V get(Object key) {
	    Node<K,V> e;
	    return (e = getNode(hash(key), key)) == null ? null : e.value;
	}
	
	static final int hash(Object key) {
	    int h;
	    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
	}

我们通过hashCode和equals方法来保证HashMap中的key值的唯一性。
当我们调用put存值时，HashMap首先会调用K的hashCode方法，获取哈希码，通过哈希码快速找到某个存放位置，这里需要注意的是，如果hashCode不同，equals一定为false，如果hashCode相同，equals不一定为true。所以理论上，hashCode可能存在冲突的情况，也就是碰撞，当碰撞发生时，计算出的hashCode是相同的，这时会比较对应hashCode位置的key，最终通过equals来比较。HashMap通过hashCode和equals最终判断出K是否已存在，如果两个hash值相等且key值相等(e.hash == hash && ((k = e.key) == key || key.equals(k))),则用新的Entry的value覆盖原来节点的value。如果两个hash值相等但key值不等 ，则将该节点插入该链表的链头。

**如果冲突发生的次数多了，链表的长度越来越长，该怎么办呢？**

随着HashMap中元素的数量越来越多，发生碰撞的概率也就越来越大，碰撞所产生的链表长度也就会越来越长，这样势必会影响HashMap的速度，因为原来找到数组的index就可以直接根据key取到值了，但是冲突严重，也就是说链表长，那就得循环链表了，时间就浪费在循环链表上了，也就慢了。为了保证HashMap的效率，系统必须要在某个临界点进行扩容处理。该临界点在当HashMap中元素的数量等于table数组长度*加载因子。但是扩容是一个非常耗时的过程，因为它需要重新计算这些数据在新table数组中的位置并进行复制处理。所以如果我们已经预知HashMap中元素的个数，那么预设元素的个数能够有效的提高HashMap的性能。


**hashMap为什么是线程不安全的？**

当hashMap扩容的时候会调用transfer方法，它是采用头插法来转移旧值到新的hashMap中去，假如转移前链表顺序是1->2->3，那么转移后就会变成3->2->1，那么在多线程的情况下就可能造成1->2的同时2->1的环形链表，进而形成死循环。

那在多线程下使用HashMap我们可以采用如下几种方案：

* 在外部包装HashMap，实现同步机制
* 使用集合的工具类 Collections的静态方法synchronizedMap，在这个方法中创建了工具类 Collections中的内部类SynchronizedMap的实例来实现HashMap的线程安全：Map m = Collections.synchronizedMap(new HashMap(...));，这里就是对HashMap做了一次包装
* 使用java.util.HashTable，效率最低
* 使用java.util.concurrent.ConcurrentHashMap，相对安全，效率较高。

**二、HashTable**

**1.1 HashTable的定义**

HashTable和HashMap很相似，但HashTable是线程安全的，同时HashTable中的key和value都不能为null。

`public class Hashtable<K,V>
    extends Dictionary<K,V>
    implements Map<K,V>, Cloneable, java.io.Serializable {}`

**1.2 Hashtable的基本方法**

	//构造一个具有默认初始容量 (11) 和默认加载因子 (0.75) 的空 Hashtable。
	Hashtable()
	//构造一个带指定初始容量和默认加载因子 (0.75) 的空 Hashtable。
	Hashtable(int initialCapacity)
	//构造一个带指定初始容量和加载因子的空 Hashtable。
	Hashtable(int initialCapacity, float loadFactor)
	//构造一个映射关系与指定 Map 相同的 Hashtable。
	Hashtable(Map<? extends K, ? extends V> t)

在Hashtable中很多操作Hashtable的方法都加上了synchronized关键字来保证线程安全，比如：get和put方法。

	public synchronized V put(K key, V value) {}
	public synchronized V get(Object key) {}
	public synchronized V remove(Object key) {}
	public synchronized void clear() {}
	public synchronized Object clone() {}

**HashMap与HashTable的几点不同**

1、HashMap是非线程安全的，而HashTable是线程安全的；

2、HashMap的遍历一般使用Iterator，而HashTable一般使用的是Enumeration。

	public interface Enumeration<E> {
	
	    boolean hasMoreElements();
	
	    E nextElement();
	}
	
	public interface Iterator<E> {
	 
	    boolean hasNext();
	
	    E next();
	
	    default void remove() {
	        throw new UnsupportedOperationException("remove");
	    }
	
	    default void forEachRemaining(Consumer<? super E> action) {
	        Objects.requireNonNull(action);
	        while (hasNext())
	            action.accept(next());
	    }
	}

从上面JDK1.8版本中的Enumeration和Iterator的代码中可以看到，Iterator对集合的操作多了一个remove，也就是说在对HashMap进行遍历的时候可以调用Iterator的remove方法来删除HashMap中的值。值得注意的是，Iterator支持fail-fast机制，在用Iterator遍历一个集合时，如果另外的线程调用了该集合的remove方法，则会抛出ConcurrentModificationException(比较了modCount == expectedModCount)，但调用Iterator的remove方法则不会。Enumeration的遍历输出是先进后出的，而Iterator的遍历输出是先进先出的。

	
**三、concurrentHashMap**

从上面可以看出Hashtable和Collections.synchronizedMap(hashMap)基本上是对读写进行加锁操作，一个线程在读写元素，其余线程必须等待，性能是十分糟糕的，而且Hashtable已经过时了。为了提高高并发下的性能，我们来看下并发安全的ConcurrentHashMap。ConcurrentHashMap之所以高效是因为它更好的降低了锁的粒度，锁加在了每个Segment上而不是直接加在整个HashMap上，参数concurrencyLevel是用户估计的并发级别，就是说你觉得最多有多少线程共同修改这个map，根据这个来确定Segment数组的大小，concurrencyLevel默认为16。ConcurrentHashMap的key和value都不能为null。

**3.1 先来看下ConcurrentHashMap 在JDK1.6中的版本。**

ConcurrentHashMap采用分段锁的机制，实现并发的更新操作，底层采用数组+链表+红黑树的存储结构。

其包含两个核心静态内部类 Segment和HashEntry。

Segment继承ReentrantLock用来充当锁的角色，每个 Segment 对象守护每个散列映射表的若干个桶。

HashEntry 用来封装映射表的键 / 值对；
每个桶是由若干个 HashEntry 对象链接起来的链表。

一个 ConcurrentHashMap 实例中包含由若干个 Segment 对象组成的数组，如图所示：

<img title="浅谈hashMap、hashTable、ConcurrentHashMap（1.6、1.8）" 图片1="" src="http://img.mukewang.com/598581d50001106005020537.png" style="width:70%" alt="ConcurrentHashMap 1.6">

**3.2 ConcurrentHashMap 在JDK1.8中的版本**

而在JDK1.8中的实现已经抛弃了Segment分段锁机制，利用CAS+Synchronized来保证并发更新的安全，底层依然采用数组+链表+红黑树的存储结构。

[CAS](http://www.jianshu.com/p/fb6e91b013cc)，Compare and Swap即比较并替换，CAS有三个操作数：内存值V、旧的预期值A、要修改的值B，当且仅当预期值A和内存值V相同时，将内存值修改为B并返回true，如果不相同则证明内存值在并发的情况下被其它线程修改过了，则不作任何修改，返回false，等待下次再修改。

table：默认为null，初始化发生在第一次插入操作，默认大小为16的数组，用来存储Node节点数据，扩容时大小总是2的幂次方。

Node：保存key，value及key的hash值的数据结构。

	static class Node<K,V> implements Map.Entry<K,V> {
	    final int hash;
	    final K key;
	    volatile V val;//保证并发的可见性
	    volatile Node<K,V> next;//保证并发的可见性
	    ...
	}

table初始化操作会延缓到第一次put行为，但put方法是可以并发的，那么如何确保table只初始化一次？在initTable方法中用sizeCtl来确保，sizeCtl变量是用volatile修饰的。

	private transient volatile int sizeCtl;
	
	public V put(K key, V value) {
	    return putVal(key, value, false);
	}
	
	/** Implementation for put and putIfAbsent */
	final V putVal(K key, V value, boolean onlyIfAbsent) {
	    if (key == null || value == null) throw new NullPointerException();
	    int hash = spread(key.hashCode());
	    int binCount = 0;
	    for (Node<K,V>[] tab = table;;) {
	        Node<K,V> f; int n, i, fh;
	        if (tab == null || (n = tab.length) == 0)
	            //初始化table
	            tab = initTable();
	        //tabAt调用Unsafe.getObjectVolatile获取指定内存的数据f
	        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
	            //如果f为null，说明table中这个位置第一次插入元素，利用casTabAt调用Unsafe.compareAndSwapObject方法插入Node节点
	            if (casTabAt(tab, i, null,
	                         new Node<K,V>(hash, key, value, null)))
	                break;                   // no lock when adding to empty bin
	        }
	        //如果hash为-1，意味有其它线程正在扩容，则一起进行扩容操作。
	        else if ((fh = f.hash) == MOVED)
	            tab = helpTransfer(tab, f);
	        //其余情况把新的Node节点按链表或红黑树的方式插入到合适的位置，这个过程采用同步内置锁实现并发
	        else {
	            V oldVal = null;
	            synchronized (f) {
	                //比较当前内存中的值是否还是f，防止被其它线程修改
	                if (tabAt(tab, i) == f) {
	                    //如果f.hash >= 0，说明f是链表结构的头结点，遍历链表，如果找到对应的node节点，则修改value，否则在链表尾部加入节点
	                    if (fh >= 0) {
	                        binCount = 1;
	                        for (Node<K,V> e = f;; ++binCount) {
	                            K ek;
	                            if (e.hash == hash &&
	                                ((ek = e.key) == key ||
	                                 (ek != null && key.equals(ek)))) {
	                                oldVal = e.val;
	                                if (!onlyIfAbsent)
	                                    e.val = value;
	                                break;
	                            }
	                            Node<K,V> pred = e;
	                            if ((e = e.next) == null) {
	                                pred.next = new Node<K,V>(hash, key,
	                                                          value, null);
	                                break;
	                            }
	                        }
	                    }
	                    else if (f instanceof TreeBin) {
	                        Node<K,V> p;
	                        binCount = 2;
	                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
	                                                       value)) != null) {
	                            oldVal = p.val;
	                            if (!onlyIfAbsent)
	                                p.val = value;
	                        }
	                    }
	                }
	            }
	            if (binCount != 0) {
	                //如果链表中节点数binCount >= TREEIFY_THRESHOLD(默认是8)，则把链表转化为红黑树结构，提高遍历查询效率。
	                if (binCount >= TREEIFY_THRESHOLD)
	                    treeifyBin(tab, i);
	                if (oldVal != null)
	                    return oldVal;
	                break;
	            }
	        }
	    }
	    //检查当前容量是否需要进行扩容
	    addCount(1L, binCount);
	    return null;
	}
	
	private final Node<K,V>[] initTable() {
	    Node<K,V>[] tab; int sc;
	    while ((tab = table) == null || tab.length == 0) {
	        //如果一个线程发现sizeCtl<0，意味着另外的线程执行CAS操作成功，当前线程只需要让出cpu时间片
	        if ((sc = sizeCtl) < 0)
	            Thread.yield(); // lost initialization race; just spin
	        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
	            try {
	                if ((tab = table) == null || tab.length == 0) {
	                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
	                    @SuppressWarnings("unchecked")
	                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
	                    table = tab = nt;
	                    sc = n - (n >>> 2);
	                }
	            } finally {
	                sizeCtl = sc;
	            }
	            break;
	        }
	    }
	    return tab;
	}
	
sizeCtl默认为0，如果ConcurrentHashMap实例化时有传参数，sizeCtl会是一个2的幂次方的值。所以执行第一次put操作的线程会执行Unsafe.compareAndSwapInt方法修改sizeCtl为-1，有且只有一个线程能够修改成功，其它线程通过Thread.yield()让出CPU时间片等待table初始化完成。

ConcurrentHashMap 是一个并发散列映射表的实现，它允许完全并发的读取，并且支持给定数量的并发更新。相比于 HashTable 和同步包装器包装的 HashMap，使用一个全局的锁来同步不同线程间的并发访问，同一时间点，只能有一个线程持有锁，也就是说在同一时间点，只能有一个线程能访问容器，这虽然保证多线程间的安全并发访问，但同时也导致对容器的访问变成串行化的了。

1.6中采用ReentrantLock 分段锁的方式，使多个线程在不同的segment上进行写操作不会发现阻塞行为；1.8中直接采用了内置锁synchronized

**参考文献：**

HashMap深度解析：http://blog.csdn.net/ghsau/article/details/16843543

谈谈HashMap线程不安全的体现：http://www.importnew.com/22011.html

JDK 1.8 ConcurrentHashMap 源码剖析：http://blog.csdn.net/lsgqjh/article/details/54867107

java中的CAS：http://www.jianshu.com/p/fb6e91b013cc
