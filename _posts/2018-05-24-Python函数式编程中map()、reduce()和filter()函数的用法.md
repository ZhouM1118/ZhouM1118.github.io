---
layout: post
title:  "Python函数式编程中map()、reduce()和filter()函数的用法"
date:   2018-05-24
desc: "Python函数式编程中map()、reduce()和filter()函数的用法"
keywords: "Python, 函数式编程, map, reduce, filter"
categories: [Python]
tags: [Python, 函数式编程, map, reduce, filter]
icon: icon-html
---

Python中`map()`、`reduce()`和`filter()`三个函数均是应用于序列的内置函数，**分别对序列进行遍历、递归计算以及过滤操作**。这三个内置函数在实际使用过程中常常和`“行内函数”lambda函数`联合使用，我们首先介绍下lambda函数。

# **1、lambda函数**

lambda函数的Python3.x API文档

> lambda
> An anonymous inline function consisting of a single expression which is evaluated when the function is called. The syntax to create a lambda function is lambda [arguments]: expression

由文档可知，lambda函数是匿名行内函数，其语法为`lambda [arguments]: expression`，比如：

```
f = lambda x, y : x * y #定义了函数f(x, y) = x * y
```

其非匿名函数表达如下：

```
def f(x, y)：
    return x * y
```

# **2、map()函数**

map()函数的Python3.x API文档

> map(function, iterable, ...)
> Return an iterator that applies function to every item of iterable, yielding the results. If additional iterable arguments are passed, function must take that many arguments and is applied to the items from all iterables in parallel. With multiple iterables, the iterator stops when the shortest iterable is exhausted. For cases where the function inputs are already arranged into argument tuples, see itertools.starmap().

`map()`函数的**输入是一个函数`function`以及一个或多个可迭代的集合`iterable`**，**在Python 2.x中`map()`函数的输出是一个集合，Python 3.x中输出的是一个迭代器**。`map()`函数主要功能为对`iterable`中的每个元素都进行`function`函数操作，并将所有的返回结果放到集合或迭代器中。**function如果是None，则其作用等同于zip()**。

例如：

```
>>> a = map(lambda x, y : x * y, range(3), range(3))
>>> b = list(a)
>>> print(b)
[0, 1, 4]
```

在Python 2.x中则不需要 `b = list(a)`，因为在Python 2.x中`map()`函数的输出直接就是一个集合。

`map()`函数的具体执行过程图如图1所示。

<img title="Python函数式编程中map()、reduce()和filter()函数的用法_" 图片1="" src="https://img.mukewang.com/5b0670f70001105b12800548.png" alt="图片描述" style="display:block; margin:auto; width:50%">

<p style="text-align:center">图 1 map()函数的具体执行过程图</p>

由图1可看出，使用map函数时，**两个可迭代的集合中的元素可以并行进行计算。**

**对于两个可迭代的集合的元素个数不一致的情况，`map()`函数会自动截断更长的那个集合，并只计算两个集合对应的元素**，比如：

```
>>> a = map(lambda x, y : x * y, range(3), range(2))
>>> b = list(a)
>>> print(b)
[0, 1]
```

# **3、reduce()函数**

`reduce()`函数的Python3.x API文档

> functools.reduce(function, iterable[, initializer])
> Apply function of two arguments cumulatively to the items of sequence, from left to right, so as to reduce the sequence to a single value. For example, reduce(lambda x, y: x+y, [1, 2, 3, 4, 5]) calculates  ((((1+2)+3)+4)+5). The left argument, x, is the accumulated value and the right argument, y, is the update value from the sequence. If the optional initializer is present, it is placed before the items of the sequence in the calculation, and serves as a default when the sequence is empty. If initializer is not given and sequence contains only one item, the first item is returned.

`reduce()`函数的**输入是一个函数`function`、一个可迭代的集合`iterable`以及一个可选的初始项`initializer`，输出为一个值**。不同于`map()`函数对序列中的元素进行逐一遍历，`reduce()`函数对序列中的元素进行递归计算。比如：

```
>>> from functools import reduce
>>> a = reduce(lambda x, y : x * y, [1, 2, 3])
>>> print(a)
6
```

在 Python3 中，`reduce()` 函数已经被从全局名字空间里移除了，它现在被放置在 `functools`模块里，如果想要使用它，则需要通过引入 `functools` 模块来调用 `reduce()` 函数。

`reduce()` 函数的具体执行过程图如图2所示。

<img title="Python函数式编程中map()、reduce()和filter()函数的用法_" 图片1="" src="https://img.mukewang.com/5b06710e0001d6dc05090439.png" alt="图片描述" style="display:block; margin:auto; width:50%">

<p style="text-align:center">图2 reduce()函数的具体执行过程图</p>

由图2可以看出，`reduce()`函数先将可迭代集合中的前两个元素进行`function`操作运算，然后将运算结果与第三个元素再进行`function`操作运算，以此类推，直到迭代完集合中所有的元素，最终返回递归结果。

# **4、filter()函数**

`filter()`函数的Python3.x API文档

> filter(function, iterable)
> Construct an iterator from those elements of iterable for which function returns true. iterable may be either a sequence, a container which supports iteration, or an iterator. If function is None, the identity function is assumed, that is, all elements of iterable that are false are removed. Note that filter(function, iterable) is equivalent to the generator expression (item for item in iterable if function(item)) if function is not Noneand (item for item in iterable if item) if function is None.
> See itertools.filterfalse() for the complementary function that returns elements of iterable for which function returns false.

`filter()`函数的**输入为一个函数`function`和一个可迭代的集合`iterable`，在Python 2.x中filter()函数的输出是一个集合，Python 3.x中输出的是一个filter类**。顾名思义，filter()函数主要是对指定可迭代集合进行过滤，筛选出集合中符合条件的元素。比如：

```
>>> a = filter(lambda x: x > 3 and x < 6, range(7))
>>> print(a)
<filter object at 0x108bf2390>
>>> b = list(a)
>>> print(b)
[4, 5]
```

# **5、map()、reduce()和filter()与for**

在Python的函数式编程中的`map()`、`reduce()`和`filter()`函数，均可用for循环来实现，那么**为什么还需要map()、reduce()和filter()函数呢？**

主要是因为Python的for命令效率不高且复杂，而`map()`、`reduce()`和`filter()`更为高效和简洁，`map()`、`reduce()`和`filter()`的循环速度比Python内置的for或while循环要快得多，其执行速度相当于C语言。

```
def demo_for():
    x = [x for x in range(100000)]
    y = [y for y in range(100000)]
    result = []
    for i in range(100000):
        result.append(x[i] + y[i])
    return result

def demo_map():
    a = map(lambda x, y: x + y, range(100000), range(100000))
    return list(a)
```

在以上的十万个元素的对比计算中，**demo_map的执行效率比demo_for的高2倍之多**。