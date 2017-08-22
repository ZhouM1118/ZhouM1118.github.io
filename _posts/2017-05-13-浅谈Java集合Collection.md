---
layout: post
title:  "浅谈Java集合Collection"
date:   2017-05-13
desc: "浅谈Java集合Collection"
keywords: "Collection"
categories: [Java]
tags: [Collection]
icon: icon-html
---

一直想找个时间系统的整理下Java的集合类，毕竟Collection包含了很多我们在实际项目中要用到的数据结构，比如列表、set、队列等，下面我们直接来看看Java集合类中各大数据结构。

Collection部分层次结构如下所示：

Collection   
├List   
│├LinkedList   
│├ArrayList   
│└Vector   
│　└Stack   
├Set   
│├HashSet   
│├TreeSet    
├Queue  
│├BlockingQueue   
│├ArrayDeque   
│└PriorityQueue   

**一、Collection**

子接口：Set，List，Queue

`public interface Collection<E> extends Iterable<E> {}`

	//获取该集合的大小
	int size();
	//判空
	boolean isEmpty();
	//判断集合是否包含某元素
	boolean contains(Object o);
	//获取集合的迭代器，迭代器的具体实现在子类中用内部类实现
	Iterator<E> iterator();
	//返回该集合的元素数组
	Object[] toArray();
	//添加元素
	boolean add(E e);
	//删除指定元素值的元素
	boolean remove(Object o);
	//判断集合是否包含指定集合中所有的元素
	boolean containsAll(Collection<?> c);
	//向当前集合添加指定集合中所有的元素
	boolean addAll(Collection<? extends E> c);
	//在当前集合中移除指定集合中所有的元素
	boolean removeAll(Collection<?> c);
	//保留指定集合中的元素，删除当前集合中不在指定集合中的元素
	boolean retainAll(Collection<?> c);
	//清空集合
	void clear();
	//Comparison and hashing
	boolean equals(Object o);
	int hashCode();
还有一些复写Iterable接口的方法

**二、List**

实现类包括：LinkedList,Vector,ArrayList。

继承Collection，可以按索引的顺序访问，元素顺序均是按添加的先后进行排列的，允许重复的元素,允许多个null元素。

除了包含Collection中的方法，还有一些自己的方法。

`public interface List<E> extends Collection<E> {}`

	//获取指定索引的元素
	E get(int index);
	//替代指定索引位置的元素
	E set(int index, E element);
	//添加指定索引位置的元素
	void add(int index, E element);
	//移除指定索引位置的元素
	E remove(int index);
	//获取指定元素在集合中第一次出现的索引值
	int indexOf(Object o);
	//获取指定元素在集合中最后一次出现的索引值
	int lastIndexOf(Object o);
	//获取ListIterator迭代器，迭代器的具体实现在子类中用内部类实现
	ListIterator<E> listIterator();
	//获取集合中从fromIndex到toIndex的值
	List<E> subList(int fromIndex, int toIndex);
	
**ListIterator**

List除了具备Collection接口的iterator方法外，还提供了listIterator，listIterator相对iterator，允许添加、设定元素，前后向遍历。

`public interface ListIterator<E> extends Iterator<E> {}`

	boolean hasNext();
	E next();
	boolean hasPrevious();
	E previous();
	int nextIndex();
	int previousIndex();
	void remove();
	void set(E e);
	void add(E e);

**2.1 ArrayList**

ArrayList依赖于数组实现的，初始长度为10的Object[]，并且可随需要而增加的动态数组。当元素超过10，采用ensureCapacity方法来扩容，ArrayList底层会新生成一个数组，长度为原来的1.5倍+1, 也就是自动增长了差不多是原来的一半，然后将原数组内容复制到新数组中，并且后续增加的内容会放到新数组中。

ArrayList对随机访问性能很好，但进行大量插入，删除操作，性能很差，因为操作之后后续元素需要移动。

`public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{}`

	private static final int DEFAULT_CAPACITY = 10;
	private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

**构造方法**

	public ArrayList(int initialCapacity) {}
	public ArrayList() {}
	public ArrayList(Collection<? extends E> c) {}
	
**扩容方法**

	public void ensureCapacity(int minCapacity) {}
	
**2.2 Vector**

特点与ArrayList相同， 不同的是Vector操作元素的方法是同步（对集合操作的方法都用synchronized关键字）的，同一时刻只能有一个线程访问，由于Vector是线程同步的，那么性能上要比ArrayList要低。Vector也是依赖数组实现的，在需要对Vector扩容时，自动增长为原来数组长度的一倍。

`public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable{}`

**构造方法  **

	//默认初始容量为10
	public Vector()  
	//默认增量为0
	public Vector(int initialCapacity)
	public Vector(int initialCapacity,int capacityIncrement)  
	        第一个参数是初始容量,第二个参数是当Vector满时的增量  
	public Vector(Collection<? extends E> c) {}

**2.2.1 Stack**

Stack类继承了Vector，实现了一个后进先出的堆栈，Stack提供了5个额外的方法实现栈的操作。

`public
class Stack<E> extends Vector<E> {}`

**构造方法**

	public Stack() {}

**实现栈的方法**

	public E push(E item) {}
	//调用了peek操作，获取栈顶元素，并删除栈顶元素
	public synchronized E pop() {}
	//获取栈顶元素，不删除栈顶元素
	public synchronized E peek() {}
	public boolean empty() {}
	public synchronized int search(Object o) {}


**2.3 LinkedList**

LinkedList功能与ArrayList，Vector相同，但内部是依赖双向链表实现的，也就是说每个结点都存放着指向前驱结点和后驱结点的引用，因此有很好的插入和删除性能,但随机访问元素的性能很差。

`public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{}`

**构造方法**  

	public LinkedList() {}
	public LinkedList(Collection<? extends E> c) {}

**LinkedList类的元素表示：Node内部类**

	private static class Node<E> {
	    E item;
	    Node<E> next;
	    Node<E> prev;
	
	    Node(Node<E> prev, E element, Node<E> next) {
	        this.item = element;
	        this.next = next;
	        this.prev = prev;
	    }
	}

**正序遍历链表**

	private static void printList(List link) {  
	    // 得到链表的迭代器,位置指向链头  
	    ListIterator li = link.listIterator();  
	    // 判断迭代器中是否有下一个元素  
	    while (li.hasNext()) {  
	        // 返回下个元素  
	        System.out.println(li.next() + " ");  
	    }  
	}  
	
**逆序遍历链表**

	private static void printReversedList(List link) {  
	    // 得到链表的迭代器,位置指向link.size()结尾  
	    ListIterator li = link.listIterator(link.size());  
	    // 判断迭代器中是否有前一个元素  
	    while (li.hasPrevious()) {  
	        // 返回前一个元素  
	        System.out.println(li.previous() + " ");  
	    }  
	}  
	
**在链表中任意位置插入元素**

链表是一个有序集合，调用LinkedList.add方法将对象插入到链表的尾部，如何添加元素到链表中的任意位置呢？这种操作可以交由迭代器负责。ListIterator的add即可实现。

	public static void addElement(){
		List<String> list = new LinkedList<String>();
		list.add("aaa");
		list.add("bbb");
		list.add("ccc");
		System.out.println(list);//[aaa, bbb, ccc]
		ListIterator<String> iterator = list.listIterator();
		iterator.next();//跳过第一个元素
		iterator.add("111");//在第二个位置插入元素111
		System.out.println(list);//[aaa, 111, bbb, ccc]
	}
	
**少用list.get(i)，用list.listIterator(i)代替**

LinkedList提供了一个用来访问某个特定元素的get方法，但这种方式的效率是非常低的，在代码中应该尽量少用这种方式，因为链表中每次查找一个元素都要从链表的头部重新开始搜索，如果索引值i大于size()/2则从列表尾端开始搜索元素。

	public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }

	Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {// >> 1 相当于除以2
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }

还是之前的例子

	System.out.println(list.get(2));//bbb
	ListIterator<String> iterator2 = list.listIterator(2);
	System.out.println(iterator2.next());//bbb

**三、Set**

不包含重复元素，最多包含一个null，元素没有顺序。

`public interface Set<E> extends Collection<E> {}`

**3.1 HashSet**

实现了Set接口，底层用hashMap来实现，hashSet的大部分方法也都是调用hashMap来实现的。散列表是用链表数组来实现的，每个列表称之为桶，当桶被占满的时候，就会出现散列冲突。散列表是无序的(hashMap和hashSet都是)。

`public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable{}`

**构造方法**

	public HashSet() {
	    map = new HashMap<>();
	}
	public HashSet(Collection<? extends E> c) {
	    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
	    addAll(c);
	}
	public HashSet(int initialCapacity, float loadFactor) {
	    map = new HashMap<>(initialCapacity, loadFactor);
	}
	public HashSet(int initialCapacity) {
	    map = new HashMap<>(initialCapacity);
	}
	HashSet(int initialCapacity, float loadFactor, boolean dummy) {
	    map = new LinkedHashMap<>(initialCapacity, loadFactor);
	}

HashSet保证了当前集合对于某一元素的唯一存在性，集合中是否存在一个对象是通过equals()和hashCode()协同判断。

HashSet的add()方法详解:  

判断已经存储在集合中的对象hashCode值是否与增加对象的hashCode值一致  
如果不一致,直接加进去  
如果一致,再进行equals()比较  
    如果equals()返回true,对象已经存在不增加进去  
    如果equals()返回false,把对象增加进去

**3.2 TreeSet**

树集合HashSet散列集很像，但树集是有序集合。在树集中可以任意插入元素到集合中，但对集合进行遍历的时候，每个值会自动地按照排序后的顺序呈现。底层实现是采用红黑树来存储元素。

添加元素的效率：hashMap > treeSet > linkedList > arrayList

treeSet中的元素如何排序的呢？

1. treeSet中的元素实现Comparable接口
1. 将Comparator对象传递给TreeSet构造器实现集合中元素的排序

**treeSet中的元素实现Comparable接口**

	public interface Comparable<T> {
	    public int compareTo(T o);
	}

比如简单的Integer

	public int compareTo(Integer anotherInteger) {
	   return compare(this.value, anotherInteger.value);
	}
	public static int compare(int x, int y) {
	   return (x < y) ? -1 : ((x == y) ? 0 : 1);
	}

如果集合中存放的是某些自定义的对象，那么需要让你实现Comparable接口，并重写compareTo方法。

**将Comparator对象传递给TreeSet构造器实现集合中元素的排序**

使用Comparable接口定义排列排序有一定的局限性，如果需要对某一特定的类的不同属性进行排序，Comparable就不能很好的满足这一需求，这个时候我们可以通过将Comparator对象传递给TreeSet构造器实现集合中元素的排序。

	public interface Comparator<T> {
		int compare(T o1, T o2);
	}

	SortedSet<Item> sortByDescription = new TreeSet<>(new Comparator<Item>(){
		public int compare(Item a, Item b){
			String descrA = a.getDescription;
			String descrB = b.getDescription;
			return descrA.compareTo(descrB);
		}
	});

**四、Queue**

**Queue接口与List、Set同一级别，都是继承了Collection接口。Queue不允许包含NULL元素。**

`public interface Queue<E> extends Collection<E> {}`

**队列方法**

	//增加一个元素，队列满则抛出IIIegaISlabEepeplian异常
	boolean add(E e);
	//添加一个元素，队列满则返回false
	boolean offer(E e);
	//移除并返回队列头部的元素，队列空则抛出一个NoSuchElementException异常
	E remove();
	//移除并返问队列头部的元素，队列空则返回null
	E poll();
	//返问但不移除队列头部的元素，队列空则抛出一个NoSuchElementException异常
	E element();
	//返问但不移除队列头部的元素，队列空则返回null
	E peek();

**4.1 BlockingQueue**

用阻塞队列两个显著的好处就是：多线程操作共同的队列时不需要额外的同步，另外就是队列会自动平衡负载，即那边（生产与消费两边）处理快了就会被阻塞掉，从而减少两边的处理速度差距。

`public interface BlockingQueue<E> extends Queue<E> {}`

**阻塞队列的方法**

	//增加一个元素，队列满则阻塞当前线程
	void put(E e) throws InterruptedException;
	//移除并返问队列头部的元素，队列空则阻塞当前线程
	E take() throws InterruptedException;
	
**4.2 ArrayDeque**

Java SE6中引入了Deque接口，提供了双端队列一系列实现。ArrayDeque中用初始容量16的数组或给定初始容量来构造一个无限双端队列。值得注意的是LinkedList也是Deque的一个实现类，也可以用来构造双端队列。

`public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable
{}`

	public ArrayDeque() {
	    elements = new Object[16];
	}
	public ArrayDeque(int numElements) {
        allocateElements(numElements);
    }

**4.3 PriorityQueue**

优先队列，队列中的元素可以按照任意的顺序输入，但却可以按照指定的顺序进行检索（TreeSet），优先队列使用了堆数据结构，堆是自我调整的二叉树，对树进行一系列的添加删除等操作，让最小（大）的元素移动到根，而不必浪费时间对元素进行排序。

和TreeSet一样，优先队列可以保存实现了Comparable接口的类对象，也可以保存在构造器中提供比较器（Comparator）的对象。

`public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {}`
    
	//优先队列默认初始值大小为11
	private static final int DEFAULT_INITIAL_CAPACITY = 11;
	
	public PriorityQueue() {
	    this(DEFAULT_INITIAL_CAPACITY, null);
	}
	public PriorityQueue(int initialCapacity) {
	    this(initialCapacity, null);
	}
	public PriorityQueue(Comparator<? super E> comparator) {
	    this(DEFAULT_INITIAL_CAPACITY, comparator);
	}

**五、Collection与Collections的区别**

java.util.Collection 是一个集合接口。它提供了对集合对象进行基本操作的通用接口方法。Collection接口在Java 类库中有很多具体的实现。Collection接口的意义是为各种具体的集合提供了最大化的统一操作方式。

java.util.Collections 是一个包装类。它包含有各种有关集合操作的静态多态方法。此类不能实例化，因为内部的构造函数被私有化了就像一个工具类，服务于Java的Collection框架。

`private Collections() {}`

**参考文献**

Java - Collection：http://blog.csdn.net/itlwc/article/details/10148321

java中queue的使用：http://www.cnblogs.com/end/archive/2012/10/25/2738493.html
