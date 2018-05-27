---
layout: post
title:  "Python数据分析工具库-Scipy 矩阵支持库"
date:   2018-05-27
desc: "Python数据分析工具库-Scipy 矩阵支持库"
keywords: "Python, 数据分析, Scipy"
categories: [Python]
tags: [Python, 数据分析, Scipy]
icon: icon-html
---

`SciPy`函数库在`NumPy`库的基础上增加了众多的数学、科学以及工程计算中常用的库函数。例如**线性代数、常微分方程数值求解、信号处理、图像处理、稀疏矩阵**等等。可以进行插值处理、信号滤波以及用C语言加速计算。


# **1、积分（scipy.integrate）**

`scipy.integrate.quad` 计算定积分

> scipy.integrate.quad(func, a, b, args=(), full_output=0, epsabs=1.49e-08, epsrel=1.49e-08, limit=50, points=None, weight=None, wvar=None, wopts=None, maxp1=50, limlst=50)

常见输入参数介绍：

 - `func`：表示定积分的函数
 - `a：float`，表示积分上限（使用`numpy.inf`表示正无穷）
 - `b：float`，表示积分下限（使用`-numpy.inf`表示负无穷）

输出参数：

 - `y：float`，从`b`到`a`函数`func`的积分
 - `abserr：float`，对结果`y`绝对误差的估计

计算定积分<img title="Python数据分析工具库-Scipy矩阵支持库_" 图片1="" src="https://img.mukewang.com/5b0a622a00018cc701200058.png" alt="图片描述" style="width:10%">

```
>>> from scipy import integrate
>>> x2 = lambda x: x**2
>>> integrate.quad(x2, 0, 4)
(21.333333333333332, 2.3684757858670003e-13)
>>> print(4**3 / 3.)  # analytical result
21.3333333333
```

计算定积分<img title="Python数据分析工具库-Scipy矩阵支持库_" 图片1="" src="https://img.mukewang.com/5b0a623a000103d301480056.png" alt="图片描述" style="width:13%">

```
>>> invexp = lambda x: np.exp(-x)
>>> integrate.quad(invexp, 0, np.inf)
(1.0, 5.842605999138044e-11)
```

**SciPy常见的积分**如下：

<img title="Python数据分析工具库-Scipy矩阵支持库" 图片1="" src="https://img.mukewang.com/5b0a624c00011cb208800934.png" alt="图片描述" style="display:block; margin:auto; width:50%">

# **2、插值（scipy.interpolate）**

**插值是在直线或曲线上的两点之间找到值的过程**。这种插值工具不仅适用于统计学，而且在科学，商业或需要预测两个现有数据点内的值时也很有用。

> class scipy.interpolate.interp1d(x, y, kind='linear', axis=-1, copy=True, bounds_error=None, fill_value=nan, assume_sorted=False)

用`x`和`y`来逼近指定函数`f：y=f(X)`。`interp1d`返回一个函数，调用该方法可使用插值来查找新点的值。

输入参数：

 - `x`：一维数组；
 - `y`：插值函数中x对应值；
 - `kind`：插值的类型，包含`‘linear’, ‘nearest’, ‘zero’, ‘slinear’, ‘quadratic’, ‘cubic’, ‘previous’, ‘next’`等。

```
>>> import matplotlib.pyplot as plt
>>> from scipy import interpolate
>>> x = np.arange(0, 10)
>>> y = np.exp(-x/3.0)
>>> f = interpolate.interp1d(x, y)
>>> xnew = np.arange(0, 9, 0.1)
>>> ynew = f(xnew)   # 使用线性插值方法返回xnew对应的插值
>>> plt.plot(x, y, 'o', xnew, ynew, ‘-‘)
>>> plt.show()
```

**result：**

<img title="Python数据分析工具库-Scipy矩阵支持库" 图片1="" src="https://img.mukewang.com/5b0a628d000136e104650288.png" alt="图片描述" style="display:block; margin:auto; width:50%">

# **3、文件操作（scipy.io）**

`scipy`的文件操作可以对`matlab、HB-format、WAV、NetCDF、IDL.sav、arff`等格式文件进行操作。

> scipy.io.loadmat(file_name, mdict=None, appendmat=True, **kwargs)

加载`matlab`文件，返回一个`dict`，`key`为变量名，`value`为加载的矩阵。

> scipy.io.savemat(file_name, mdict, appendmat=True, format='5', long_field_names=False, do_compression=False, oned_as='row')

保存`mat`格式文件，`file_name`为保存文件名，`mdict`为保存的键值对。

```
from scipy import io as spio
>>>spio.savemat("test.mat", {'a':a})
```

读取图片文件 导包：`from scipy import misc` 读取：`data = misc.imread("123.png”)` 

与`matplotlib`中`plt.imread('fname.png')`类似 ；执行`misc.imread`时可能提醒不存在这个模块，那就安装`pillow`的包。

# **4、scipy中的线性代数（scipy.linalg）**

最好结合矩阵分析或线性代数课程书一起看

`numpy`中也有线性代数库`Linear algebra (numpy.linalg)`，但是推荐用`scipy.linalg`代替`numpy.linalg`，`scipy.linalg`中包含大部分`numpy.linalg`中的函数，且在同名函数中包含更多的功能。

**求矩阵的逆**

> scipy.linalg.inv(a, overwrite_a=False, check_finite=True)

```
>>> from scipy import linalg
>>> a = np.array([[1., 2.], [3., 4.]])
>>> linalg.inv(a)
array([[-2. ,  1. ],
       [ 1.5, -0.5]])
```

如果矩阵`a`不可逆，则会抛异常`LinAlgError: singular matrix`

**求解线性方程组Ax=b**

> scipy.linalg.solve(a, b, sym_pos=False, lower=False, overwrite_a=False, overwrite_b=False, debug=None, check_finite=True, assume_a='gen', transposed=False)

```
>>> a = np.array([[3, 2, 0], [1, -1, 0], [0, 5, 1]])
>>> b = np.array([2, 4, -1])
>>> from scipy import linalg
>>> x = linalg.solve(a, b)
>>> x
array([ 2., -2.,  9.])
```

**Eigenvalue Problems 求特征值及特征向量**

**奇异值分解**

> scipy.linalg.svd(a, full_matrices=True, compute_uv=True, overwrite_a=False, check_finite=True, lapack_driver='gesdd’)

```
>>> from scipy import linalg
>>> m, n = 9, 6
>>> a = np.random.randn(m, n) + 1.j*np.random.randn(m, n)
>>> U, s, Vh = linalg.svd(a)
>>> U.shape,  s.shape, Vh.shape
((9, 9), (6,), (6, 6))
```

# **5、优化算法（scipy.optimize）**

 该模块包含以下几个算法：

 - 使用各种算法(例如`BFGS`，`Nelder-Mead`单纯形，牛顿共轭梯度，`COBYLA`或`SLSQP`)的无约束和约束最小化多元标量函数`minimize()`
   全局(蛮力)优化程序(例如，`anneal()，basinhopping()`)
 - 最小二乘最小化(`leastsq()`)和曲线拟合(`curve_fit()`)算法
 - 标量单变量函数最小化(`minim_scalar()`)和根查找(`newton()`)
 - 使用多种算法(例如，`Powell`，`Levenberg-Marquardt`混合或`Newton-Krylov`等大规模方法)的多元方程系统求解

# **6、统计函数（scipy.stats）**

所有的统计函数都位于子包`scipy.stats`中，该模块包含大量的概率分布以及不断增长的统计函数库。

**T-检验stats.ttest_ind**

> scipy.stats.ttest_ind(a, b, axis=0, equal_var=True, nan_policy='propagate’)

```
>>> from scipy import stats
>>> np.random.seed(12345678)
>>> rvs1 = stats.norm.rvs(loc=5,scale=10,size=500)
>>> rvs2 = stats.norm.rvs(loc=5,scale=10,size=500)
>>> stats.ttest_ind(rvs1,rvs2)
(0.26833823296239279, 0.78849443369564776)
```

输出第一个参数为`T-检验值`，第二个为概率`p`: 两个过程相同的概率。如果其值接近1，那么两个过程几乎可以确定是相同的，如果其值接近0，那么它们很可能拥有不同的均值。

# **参考文献**

**SciPy API文档**：https://docs.scipy.org/doc/scipy/reference/index.html

**SciPy教程**：https://www.yiibai.com/scipy/





