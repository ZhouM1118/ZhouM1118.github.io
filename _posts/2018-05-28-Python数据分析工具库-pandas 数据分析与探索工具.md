---
layout: post
title:  "Python数据分析工具库-pandas 数据分析与探索工具"
date:   2018-05-28
desc: "Python数据分析工具库-pandas 数据分析与探索工具"
keywords: "Python, 数据分析, pandas"
categories: [Python]
tags: [Python, 数据分析, pandas]
icon: icon-html
---

**pandas是基于numpy的一个高级数据结构和操作的数据分析与探索工具**，本文基于pandas API文档对pandas的**两个重要的数据结构、基本函数、函数应用、排序以及层次化索引**进行分析，对于本文的示例代码做如下约定：

```
import numpy as np
from pandas import Series, DataFrame
import pandas as pd
```

# **1 pandas 数据结构**

## **1.1 Series**

**Series是一种类似于一维数组的对象**，它的索引由参数index指定，创建一个Series的语法如下：

> s = Series(data, index=index)


data的类型可以是：**Python的dict、numpy的ndarray或标量值**

**Python的dict**

```
>>> d = {'b' : 1, 'a' : 0, 'c' : 2}
>>> pd.Series(d)
b    1
a    0
c    2
dtype: int64
```

**numpy的ndarray**

```
>>> s = pd.Series(np.random.randn(5), index=['a', 'b', 'c', 'd', 'e'])
>>> s
a    0.4691
b   -0.2829
c   -1.5091
d   -1.1356
e    1.2121
dtype: float64
```

**标量值**

```
>>> pd.Series(5., index=['a', 'b', 'c', 'd', 'e'])
a    5.0
b    5.0
c    5.0
d    5.0
e    5.0
dtype: float64
```

**如果不指定index的值，会自动创建一个0~N-1（N为数据的长度）的整数型索引**。

与ndarray类似，你也可以对Series进行**切片、索引**等操作，如下：

```
>>> s
a    0.4691
b   -0.2829
c   -1.5091
d   -1.1356
e    1.2121

>>> s[0]
0.4691

>>> s[:3]
a    0.4691
b   -0.2829
c   -1.5091
dtype: float64

>>> s[s > s.median()]
a    0.4691
e    1.2121
dtype: float64

>>> s[[4, 3, 1]]
e    1.2121
d   -1.1356
b   -0.2829
dtype: float64

>>> np.exp(s)
a    1.5986
b    0.7536
c    0.2211
d    0.3212
e    3.3606
dtype: float64
```

同时也可以**将Series看出是一个定长的有序字典**，如下：

```
>>> s['a’]
0.4691

>>> s['e'] = 12.

>>> s
a     0.4691
b    -0.2829
c    -1.5091
d    -1.1356
e    12.0000
dtype: float64

>>> 'e' in s
True

>>> 'f' in s
False
```

Series和ndarray之间的一个关键区别是，**Series在算术运算中会自动对齐不同索引的数据**。因此，我们在操作Series时不必考虑Series是否具有相同的标签，比如：

```
>>> s[1:] + s[:-1]
a       NaN
b   -0.5657
c   -3.0181
d   -2.2713
e       NaN
dtype: float64
```

在pandas中，**如果一个索引在一个Series中而不存在另一个Series中，那么它们操作的结果用NaN（即非数字，not a number）来表示**，在pandas中常用它来表示缺失或NA值，pandas的isnull和notnull函数可以来检测Series中的缺失数据。

```
>>> pd.isnull(s) # 等同于s.isnull()
a   True
b   False
c   False
d   False
e   True

>>> pd.notnull(s) # 等同于s.notnull()
a   False
b   True
c   True
d   True
e   False
```

## **1.2 DataFrame**

**DataFrame是一个表格型的数据结构，它包含一组有序的列，每列可以是不同类型的值。DataFrame既有行索引也有列索引，可看成是由Series组成的字典。**

创建DataFrame的方式有很多种，下图展示了可以输给DataFrame构造器的数据：

<img title="Python数据分析工具库-pandas数据分析与探索工具_" 图片1="" src="https://img.mukewang.com/5b0bfc5d0001f23c18681264.png" alt="图片描述" style="display:block; margin:auto; width:50%">

其中最常见的是**直接传入一个由等长列表或numpy数组组成的字典**：

```
>>> d = {'one' : [1., 2., 3., 4.],
   ....:      'two' : [4., 3., 2., 1.]}
   ....: 
>>> pd.DataFrame(d)
   one  two
0  1.0  4.0
1  2.0  3.0
2  3.0  2.0
3  4.0  1.0
```

或者**由Series组成的字典**

```
>>> d = {'one' : Series([1., 2., 3.], index=['a', 'b', 'c']),
   ....:      'two' : Series([1., 2., 3., 4.], index=['a', 'b', 'c', 'd'])}
   ....: 
>>> df = DataFrame(d)
>>> df
   one  two
a  1.0  1.0
b  2.0  2.0
c  3.0  3.0
d  NaN  4.0
```

或者**由字典组成的列表**

```
>>> data2 = [{'a': 1, 'b': 2}, {'a': 5, 'b': 10, 'c': 20}]

>>> DataFrame(data2)
   a   b     c
0  1   2   NaN
1  5  10  20.0

>>> df = DataFrame(data2, index=['first', 'second’])
>>> df
        a   b     c
first   1   2   NaN
second  5  10  20.0
```

**处理时间序列数据**

```
>>> index = pd.date_range('1/1/2000', periods=8)
>>> df = pd.DataFrame(np.random.randn(8, 3), index=index, columns=list('ABC'))
>>> df
                 A       B       C
2000-01-01 -1.2268  0.7698 -1.2812
2000-01-02 -0.7277 -0.1213 -0.0979
2000-01-03  0.6958  0.3417  0.9597
2000-01-04 -1.1103 -0.6200  0.1497
2000-01-05 -0.7323  0.6877  0.1764
2000-01-06  0.4033 -0.1550  0.3016
2000-01-07 -2.1799 -1.3698 -0.9542
2000-01-08  1.4627 -1.7432 -0.8266
```

**创建bool类型的DataFrame**

```
>>> df1 = pd.DataFrame({'a' : [1, 0, 1], 'b' : [0, 1, 1] }, dtype=bool)
>>> df2 = pd.DataFrame({'a' : [0, 1, 1], 'b' : [1, 1, 0] }, dtype=bool)
>>> df1 & df2
       a      b
0  False  False
1  False   True
2   True  False
>>> df1 | df2
      a     b
0  True  True
1  True  True
2  True  True
>>> df1 ^ df2
       a      b
0   True   True
1   True  False
2  False   True
>>> -df1
       a      b
0  False   True
1   True  False
2  False  False
```

**DaraFrame的转置**：

```
>>> df.T
     first    second
a     1            5
b     2            10
c     NaN      20.0
```
**DataFrame的索引**

和Series类似，**DataFrame的索引方式**有：

<img title="Python数据分析工具库-pandas数据分析与探索工具_" 图片1="" src="https://img.mukewang.com/5b0bfcc30001a4f410360250.png" alt="图片描述" style="display:block; margin:auto; width:50%">

```
>>> df
   one  two
a  1.0  1.0
b  2.0  2.0
c  3.0  3.0
d  NaN  4.0
>>> df['one’] # 列索引，等同于df.one
a    1.0
b    2.0
c    3.0
d    NaN
```

**对DataFrame进行索引其实是获取一个或多个列，但对DataFrame进行切片或布尔型数组时选取的是行**：

```
>>> df[:2]
   one  two
a  1.0  1.0
b  2.0  2.0
>>> df[df[‘one’] < 3]
   one  two
a  1.0  1.0
b  2.0  2.0
```

**在DataFrame的行上进行标签索引的方式**

```
>>> df
   one  bar   flag  foo  one_trunc
a  1.0  1.0  False  bar        1.0
b  2.0  2.0  False  bar        2.0
c  3.0  3.0   True  bar        NaN
d  NaN  NaN  False  bar        NaN 
>>> df.loc['b’]  # 等同于df.iloc[1]、df.ix[‘b’]、df.ix[1]
one              2
bar              2
flag         False
foo            bar
one_trunc        2
Name: b, dtype: object
```

**通过索引方式返回的列是相应数据的视图（view），不是副本（copy），故对返回的Series所做的任何就地修改全部会反映到源DataFrame上。**

## **1.3 重新索引reindex**

> Series.reindex(index=None, **kwargs)

调用reindex方法将会根据新索引对Series或DataFrame进行重新索引，**reindex函数的参数**如下图所示。

<img title="Python数据分析工具库-pandas数据分析与探索工具_" 图片1="" src="https://img.mukewang.com/5b0bfcd90001407b18600664.png" alt="图片描述" style="display:block; margin:auto; width:50%">

如果某个索引值在当前数据结构不存在，那么用缺失值NaN填充，当然也可以**使用fill_value参数来指定填充的值**，或**用mothod参数来指定填充的方式**（填充的方式有ffill：前向填充值；bfill：后向填充值），下图展示了mothod参数可选的参数。

<img title="Python数据分析工具库-pandas数据分析与探索工具_" 图片1="" src="https://img.mukewang.com/5b0bfce90001041507280180.png" alt="图片描述" style="display:block; margin:auto; width:50%">


```
>>> rng = pd.date_range('1/3/2000', periods=8)

>>> ts = pd.Series(np.random.randn(8), index=rng)

>>> ts2 = ts[[0, 3, 6]]

>>> ts
2000-01-03    0.466284
2000-01-04   -0.457411
2000-01-05   -0.364060
2000-01-06    0.785367
2000-01-07   -1.463093
2000-01-08    1.187315
2000-01-09   -0.493153
2000-01-10   -1.323445
Freq: D, dtype: float64

>>> ts2
2000-01-03    0.466284
2000-01-06    0.785367
2000-01-09   -0.493153
dtype: float64

>>> ts2.reindex(ts.index)
2000-01-03    0.466284
2000-01-04         NaN
2000-01-05         NaN
2000-01-06    0.785367
2000-01-07         NaN
2000-01-08         NaN
2000-01-09   -0.493153
2000-01-10         NaN
Freq: D, dtype: float64

>>> ts2.reindex(ts.index, method='ffill')
2000-01-03    0.466284
2000-01-04    0.466284
2000-01-05    0.466284
2000-01-06    0.785367
2000-01-07    0.785367
2000-01-08    0.785367
2000-01-09   -0.493153
2000-01-10   -0.493153
Freq: D, dtype: float64

>>> ts2.reindex(ts.index, method='bfill')
2000-01-03    0.466284
2000-01-04    0.785367
2000-01-05    0.785367
2000-01-06    0.785367
2000-01-07   -0.493153
2000-01-08   -0.493153
2000-01-09   -0.493153
2000-01-10         NaN
Freq: D, dtype: float64

>>> ts2.reindex(ts.index, method='nearest')
2000-01-03    0.466284
2000-01-04    0.466284
2000-01-05    0.785367
2000-01-06    0.785367
2000-01-07    0.785367
2000-01-08   -0.493153
2000-01-09   -0.493153
2000-01-10   -0.493153
Freq: D, dtype: float64
```

**为DataFrame添加列**（修改也是类似思路，将列表或数组赋值给某个列时，要求其长度必须与DataFrame的长度相匹配；而如果赋值的是一个Series，会自动精确匹配DataFrame的索引，空位将自动被填上缺失值）：

```
>>> df['three'] = df['one'] * df['two’] # 添加列，列索引为“three”
>>> df['flag'] = df['one'] > 2  # 添加列，列索引为“flag”
>>> df
   one  two  three   flag
a  1.0  1.0    1.0  False
b  2.0  2.0    4.0  False
c  3.0  3.0    9.0   True
d  NaN  4.0    NaN  False
>>> val=Series([1.2, 1.5, 1.7], index=[‘a’, ‘c’, ‘d’])
>>> df[‘debt’] = val
>>> df
   one  two  three   flag debt
a  1.0  1.0    1.0  False    1.2
b  2.0  2.0    4.0  False    NaN
c  3.0  3.0    9.0   True    1.5
d  NaN  4.0    NaN  False  1.7
```

也可以**在指定位置插入列**：

```
>>> df.insert(1, 'bar', df['one'])
>>> df
   one  bar  two  three   flag                 
a  1.0  1.0  1.0    1.0    False 
b  2.0  2.0  2.0    4.0   False  
c  3.0  3.0  3.0    9.0   True  
d  NaN  NaN 4.0    NaN  False  
```

我们也可以**删除列**：

```
>>> del df['two’] # 删除列索引为“two”的列
>>> three = df.pop('three’) # 删除列索引为“three”的列，等同于df.drop(’three’, axis=1)
>>> df
   one   flag
a  1.0  False
b  2.0  False
c  3.0   True
d  NaN  False
```

**删除行**

```
>>> df.drop([‘c’, ‘d'])
>>> df
    one   flag
a  1.0  False
b  2.0  False
```

## **1.4 广播（broadcasting）**

```
>>> df = DataFrame({'one' : pd.Series(np.random.randn(3), index=['a', 'b', 'c']), # 构造DataFrame
   ....:                    'two' : pd.Series(np.random.randn(4), index=['a', 'b', 'c', 'd']),
   ....:                    'three' : pd.Series(np.random.randn(3), index=['b', 'c', 'd'])})

>>> df
        one       two     three
a -1.101558  1.124472       NaN
b -0.177289  2.487104 -0.634293
c  0.462215 -0.486066  1.931194
d       NaN -0.456288 -1.222918

>>> row = df.iloc[1] # 取出第二行

>>> column = df['two’] #取出第二列

>>> df.sub(row, axis='columns’) #广播，每一行都减去第二行，等同于df - row
        one       two     three
a -0.924269 -1.362632       NaN
b  0.000000  0.000000  0.000000
c  0.639504 -2.973170  2.565487
d       NaN -2.943392 -0.588625

>>> df.sub(row, axis=1) 
        one       two     three
a -0.924269 -1.362632       NaN
b  0.000000  0.000000  0.000000
c  0.639504 -2.973170  2.565487
d       NaN -2.943392 -0.588625

>>> df.sub(column, axis='index’) # 广播，每一列都减去第二列
        one  two     three
a -2.226031  0.0       NaN
b -2.664393  0.0 -3.121397
c  0.948280  0.0  2.417260
d       NaN  0.0 -0.766631

>>> df.sub(column, axis=0)
        one  two     three
a -2.226031  0.0       NaN
b -2.664393  0.0 -3.121397
c  0.948280  0.0  2.417260
d       NaN  0.0 -0.766631
```

# **2 pandas基本函数**

```
>>> df
        one       two     three
a -1.101558  1.124472       NaN
b -0.177289  2.487104 -0.634293
c  0.462215 -0.486066  1.931194
d       NaN -0.456288 -1.222918
```

pandas包含丰富的函数来对数据进行统计分析。

## **2.1 mean函数**

```
>>> df.mean(0) #对DataFrame的每列求平均数，axis=0
one     -0.272211
two      0.667306
three    0.024661
dtype: float64

>>> df.mean(1) #对DataFrame的每行求平均数，axis=1
a    0.011457
b    0.558507
c    0.635781
d   -0.839603
dtype: float64
```

**pandas中的axis**

我们来分析下pandas中的axis，它究竟是代表行还是列呢？以上的mean的参数axis=1时，但在第一节中的drop函数时却删除了一列，那么到底axis=1代表的是行还是列呢？df.mean其实是在每一行上取所有列的均值，而不是保留每一列的均值。也许简单的来记就是axis=0代表往跨行（down)，而axis=1代表跨列（across)，即**当axis=0时表示沿着每一列或行索引值向下执行方法；当axis=1时表示沿着每一行或列索引值模向执行对应的方法**。下图形象解释了axis的含义。

<img title="Python数据分析工具库-pandas数据分析与探索工具_" 图片1="" src="https://img.mukewang.com/5b0bff9e0001f97706520341.jpg" alt="图片描述" style="display:block; margin:auto; width:50%">

## **2.2 sum函数**

```
>>> df.sum(0, skipna=False) # 按列求和，axis=0，skipna参数表示是否排除缺失值，默认为True
one           NaN
two      2.669223
three         NaN
dtype: float64

>>> df.sum(axis=1) # 按列求和，axis=0
a    0.022914
b    1.675522
c    1.907343
d   -1.679206
dtype: float64
```

## **2.3 pandas常见的统计分析函数**

<img title="Python数据分析工具库-pandas数据分析与探索工具_" 图片1="" src="https://img.mukewang.com/5b0bffb00001792007580776.png" alt="图片描述" style="display:block; margin:auto; width:50%">

# **3、函数应用**

## **3.1 apply函数**

apply函数可以将一个函数应用在DataFrame的某个轴上。

> DataFrame.apply(func, axis=0, broadcast=None, raw=False, reduce=None, result_type=None, args=(), **kwds)

```
>>> df.apply(np.mean, axis=1)# 等同于df.apply(‘mean’, axis=1)
a    0.011457
b    0.558507
c    0.635781
d   -0.839603
dtype: float64

>>> df.apply(lambda x: x.max() - x.min())
one      1.563773
two      2.973170
three    3.154112
dtype: float64
```

**元素级函数应用可以使用applymap()**

## **3.2 agg()**

在pandas V0.20.0版本中新出现一个函数**agg()**（DataFrame.aggregate()的简版），**agg()与apply()不同的是，其在指定的轴上可以使用一个或多个函数对数据进行聚合操作。**

> DataFrame.agg(func, axis=0, *args, **kwargs)

```
>>> tsdf = pd.DataFrame(np.random.randn(10, 3), columns=['A', 'B', 'C'],
   .....:                     index=pd.date_range('1/1/2000', periods=10))

>>> tsdf
                   A         B         C
2000-01-01  0.170247 -0.916844  0.835024
2000-01-02  1.259919  0.801111  0.445614
2000-01-03  1.453046  2.430373  0.653093
2000-01-04       NaN       NaN       NaN
2000-01-05       NaN       NaN       NaN
2000-01-06       NaN       NaN       NaN
2000-01-07       NaN       NaN       NaN
2000-01-08 -1.874526  0.569822 -0.609644
2000-01-09  0.812462  0.565894 -1.461363
2000-01-10 -0.985475  1.388154 -0.078747

>>> tsdf.agg(np.sum) # 一个函数的时候等同于tsdf.apply(np.sum)
A    0.835673
B    4.838510
C   -0.216025
dtype: float64

>>> tsdf.agg(['sum', 'mean’]) # 对多个函数进行聚合操作
             A         B         C
sum   0.835673  4.838510 -0.216025
mean  0.139279  0.806418 -0.036004
```

## **3.3 transform()**

transform()与agg()函数很像。

> DataFrame.transform(func, *args, **kwargs)

```
>>> tsdf = pd.DataFrame(np.random.randn(10, 3), columns=['A', 'B', 'C'],
   .....:                     index=pd.date_range('1/1/2000', periods=10))
>>> tsdf
                   A         B         C
2000-01-01 -0.578465 -0.503335 -0.987140
2000-01-02 -0.767147 -0.266046  1.083797
2000-01-03  0.195348  0.722247 -0.894537
2000-01-04       NaN       NaN       NaN
2000-01-05       NaN       NaN       NaN
2000-01-06       NaN       NaN       NaN
2000-01-07       NaN       NaN       NaN
2000-01-08 -0.556397  0.542165 -0.308675
2000-01-09 -1.010924 -0.672504 -1.139222
2000-01-10  0.354653  0.563622 -0.365106

>>> tsdf.transform([np.abs, lambda x: x+1])
                   A                   B                   C          
            absolute  <lambda>  absolute  <lambda>  absolute  <lambda>
2000-01-01  0.578465  0.421535  0.503335  0.496665  0.987140  0.012860
2000-01-02  0.767147  0.232853  0.266046  0.733954  1.083797  2.083797
2000-01-03  0.195348  1.195348  0.722247  1.722247  0.894537  0.105463
2000-01-04       NaN       NaN       NaN       NaN       NaN       NaN
2000-01-05       NaN       NaN       NaN       NaN       NaN       NaN
2000-01-06       NaN       NaN       NaN       NaN       NaN       NaN
2000-01-07       NaN       NaN       NaN       NaN       NaN       NaN
2000-01-08  0.556397  0.443603  0.542165  1.542165  0.308675  0.691325
2000-01-09  1.010924 -0.010924  0.672504  0.327496  1.139222 -0.139222
2000-01-10  0.354653  1.354653  0.563622  1.563622  0.365106  0.634894
```

# **4、排序**

根据设置的条件对数据集进行排序（sorting），是pandas的一个重要的内置运算，sort_index可以对行或列索引进行排序，并返回一个已排序的新对象。

> Series.sort_index(axis=0, level=None, ascending=True, inplace=False, kind='quicksort', na_position='last', sort_remaining=True)
> DataFrame.sort_index(axis=0, level=None, ascending=True, inplace=False, kind='quicksort', na_position='last', sort_remaining=True, by=None)

```
>>> df = pd.DataFrame({'one' : pd.Series(np.random.randn(3), index=['a', 'b', 'c']),
   .....:                    'two' : pd.Series(np.random.randn(4), index=['a', 'b', 'c', 'd']),
   .....:                    'three' : pd.Series(np.random.randn(3), index=['b', 'c', 'd'])})

>>> unsorted_df = df.reindex(index=['a', 'd', 'c', 'b'],
   .....:                          columns=['three', 'two', 'one'])
   .....: 

>>> unsorted_df
      three       two       one
a       NaN  0.708543  0.036274
d -0.540166  0.586626       NaN
c  0.410238  1.121731  1.044630
b -0.282532 -2.038777 -0.490032

>>> unsorted_df.sort_index() #axis=0，对行轴的索引进行排序
      three       two       one
a       NaN  0.708543  0.036274
b -0.282532 -2.038777 -0.490032
c  0.410238  1.121731  1.044630
d -0.540166  0.586626       NaN

>>> unsorted_df.sort_index(axis=1)
        one     three       two
a  0.036274       NaN  0.708543
d       NaN -0.540166  0.586626
c  1.044630  0.410238  1.121731
b -0.490032 -0.282532 -2.038777
```

数据默认是按升序排序，可以通过设置参数**ascending=False**进行降序排序。

```
>>> unsorted_df.sort_index(ascending=False)
      three       two       one
d -0.540166  0.586626       NaN
c  0.410238  1.121731  1.044630
b -0.282532 -2.038777 -0.490032
a       NaN  0.708543  0.036274
```

**按元素值进行排序**

```
>>> df1 = pd.DataFrame({'one':[2,1,1,1],'two':[1,3,2,4],'three':[5,4,3,2]})

>>> df1.sort_values(by='two')
   one  two  three
0    2    1      5
2    1    2      3
1    1    3      4
3    1    4      2
```

**对多个列进行排序**

```
>>> df1[['one', 'two', 'three']].sort_values(by=['one','two’]) # 在one列有序的基础上再对two列进行排序
   one  two  three
2    1    2      3
1    1    3      4
3    1    4      2
0    2    1      5
```

# **5、层次化索引（hierarchical indexing）**

层次化索引是pandas的一个重要功能，层次化索引可以在一个轴上有多个索引级别，并以低维度形式处理高纬度数据。

```
>>> arrays = [np.array(['bar', 'bar', 'baz', 'baz', 'foo', 'foo', 'qux', 'qux']),
   ....:           np.array(['one', 'two', 'one', 'two', 'one', 'two', 'one', 'two'])]

>>> s = pd.Series(np.random.randn(8), index=arrays)

>>> s
bar  one   -0.861849
     two   -2.104569
baz  one   -0.494929
     two    1.071804
foo  one    0.721555
     two   -0.706771
qux  one   -1.039575
     two    0.271860
dtype: float64

>>> df = pd.DataFrame(np.random.randn(3, 8), index=['A', 'B', 'C'], columns=index)

>>> df
first                     bar                     baz                   foo                     qux          
second       one       two       one       two       one       two       one       two
A       0.895717  0.805244 -1.206412  2.565646  1.431256  1.340309 -1.170299 -0.226169
B       0.410835  0.813850  0.132003 -0.827317 -0.076467 -1.187678  1.130127 -1.436737
C      -1.413681  1.607920  1.024180  0.569605  0.875906 -2.211372  0.974466 -2.006747
```

**索引方式**

```
>>> df['bar']
second       one       two
A       0.895717  0.805244
B       0.410835  0.813850
C      -1.413681  1.607920

>>> df['bar', 'one']
A    0.895717
B    0.410835
C   -1.413681
Name: (bar, one), dtype: float64

>>> df.xs('one', level='second', axis=1)
first       bar       baz       foo       qux
A      0.895717 -1.206412  1.431256 -1.170299
B      0.410835  0.132003 -0.076467  1.130127
C     -1.413681  1.024180  0.875906  0.974466

>>> df.xs(('one', 'bar'), level=('second', 'first'), axis=1)
first        bar
second       one
A       0.895717
B       0.410835
C      -1.413681
```
