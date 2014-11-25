---
title: "not-so-obvious Python stuff"
date: 2014-07-20 21:13
---

近期开始整理以前看过的一些非常好的python文章, 这篇文章就会大概讲一讲Python里面的有些小坑.

## MRO

当用到多继承的时候, 在父类中去查找继承的方法的顺序叫做 **Method Resolution Order**. Python里面属性的查找也会是这样.

Python中老式的类(classic classes)的MRO为 **depth-first left-to-right**, 就是说是从左往右深度遍历. 举个例子:

    In [1]: class A:
       ...:     def name(self):
       ...:         print 'class A'
       ...:

    In [2]: class B(A):
       ...:     pass
       ...:

    In [3]: class C(A):
       ...:     def name(self):
       ...:         print 'class C'
       ...:

    In [4]: class D(B, C):
       ...:     pass

`D().name()`打印了`class A`, 它的MRO为: D B A C A. 本来我是想让D继承B, C, 如果B里面没有name方法, 那应该是查找平级的C中的name方法. 但是由于是深度遍历, 所以调用的是A的name方法. 虽然这种多继承的情况比较少见. 但是对于新式类(new-style classes)来说, 就特别普遍了(都继承了`object`).

为了解决这个坑, 新式类在深度遍历的时候, 如果有重复的字段, 除了最后一次出现的字段的, 删掉其他重复的字段.所以在新式类中找出上述例子中D的name方法时:

    In [13]: D.__mro__
    Out[13]: (__main__.D, __main__.B, __main__.C, __main__.A, object)

> 更多关于MRO的信息请看[这篇文章](http://python-history.blogspot.ru/2010/06/method-resolution-order.html).

## List操作

直接上代码:

    In [1]: a = [1, 2]

    In [2]: id(a)
    Out[2]: 4452270664

    In [3]: a += [3]

    In [4]: id(a)
    Out[4]: 4452270664

    In [5]: a = a + [4]  # 只有+的时候id会改变

    In [6]: a
    Out[6]: [1, 2, 3, 4]

    In [7]: id(a)
    Out[7]: 4452271096

    In [8]: a.append(5)

    In [9]: id(a)
    Out[9]: 4452271096

    In [10]: a.extend([6])

    In [11]: id(a)
    Out[11]: 4452271096

对列表的 `+`, `+=`, `append`, `extend` 操作中, 只有 `+` 操作会改变列表的id.

## is 和 ==

总体来说 `is` 是判断2者身份(id), 而 `==` 是判断的值.

    In [1]: a = 1

    In [2]: b = 1

    In [3]: a is b
    Out[3]: True

    In [4]: a == b
    Out[4]: True

    In [5]: c = 257

    In [6]: d = 257

    In [7]: c == d
    Out[7]: True

    In [8]: c is d
    Out[8]: False

    In [9]: e = -6

    In [10]: f = -6

    In [11]: e is f
    Out[11]: False

    In [12]: e == f
    Out[12]: True

python缓存了-5 到 256的对象, 所以在这个之间的相同的数, `is`返回的都会是 `True` , 但是不在这个返回就是2个对象所以 `is` 返回的是 `False` , 但是 `==` 总是返回 `True` .

字符串也有缓存, 对只含有 **字母** , **数字** , **下划线** 组成的字符串来说, 相同的字符串 `is` 和 `==` 返回的是 `True` 如:

    In [1]: a = 'hello'

    In [2]: b = 'hello'

    In [3]: a is b
    Out[3]: True

    In [4]: a == b
    Out[4]: True

如果包含了其他字符, 则 `is` 返回 `False` , `==` 为 `True` :

    In [1]: a = '!a'

    In [2]: b = '!a'

    In [3]: a is b
    Out[3]: False

    In [4]: a = 'hello world'

    In [5]: b = 'hello world'

    In [6]: a is b
    Out[6]: False

但是也有特例:

    In [1]: a = float('nan')

    In [2]: a is a
    Out[2]: True

    In [3]: a == a
    Out[3]: False

哈哈哈哈. `a` 啥也不等于, 连他自己都不等于, 是不是很郁闷?

如果要判断是不是 `nan`:

    In [1]: import math

    In [2]: a = float('nan')

    In [3]: a
    Out[3]: nan

    In [4]: math.isnan(a)
    Out[4]: True

判断是不是无限大:

    In [1]: a = float('Inf')

    In [2]: a > 100000
    Out[2]: True

    In [3]: import math

    In [4]: math.isinf(a)
    Out[4]: True

## 浅拷贝 深拷贝

先来看下面的例子:

    In [1]: a = [1, 2]

    In [2]: id(a)
    Out[2]: 4415456176

    In [3]: b = a

    In [4]: id(b)
    Out[4]: 4415456176

    In [5]: c = a[:]

    In [6]: id(c)
    Out[6]: 4415465592

    In [7]: d = list(a)

    In [8]: id(d)
    Out[8]: 4415456608

    In [9]: import copy

    In [10]: e = copy.copy(a)

    In [11]: id(e)
    Out[11]: 4415435048

直接赋值很好理解, b的id显然应该和a是一致的, 但是通过切片的方式或者工厂函数以及通过 `copy` 函数创建的c, d和e却和a有着不同的id. 实际上通过 **切片** , **工厂函数** 或者 `copy` 函数的方式来拷贝对象, 就是一种 **浅拷贝** . 浅拷贝只拷贝父对象.

再来看一个例子:

    In [12]: obj = ['name',['age',18]]

    In [13]: a=obj[:]

    In [14]: b=list(obj)

    In [15]: for x in obj,a,b:
       ....:         print id(x)
       ....:
    4415437424
    4415437712
    4415435768

    In [16]: a[0] = 'lisi'

    In [17]: b[0] = 'zhangsan'

    In [18]:

    In [18]: print a
    ['lisi', ['age', 18]]

    In [19]: print b
    ['zhangsan', ['age', 18]]

    In [20]:

    In [20]: a[1][1] = 25

    In [21]:

    In [21]: print a
    ['lisi', ['age', 25]]

    In [22]: print b
    ['zhangsan', ['age', 25]]

在这个例子中, 修改a和b的名字都不会互相影响, 单只是修改了a的年龄, 为什么就改变b中的年龄呢?

实际上, 我们创建的a与b都是从obj对象的浅拷贝, obj中第一个元素是字符串属于不可变类型(immutable), 第二个元素是列表属于可变类型(mutable). 因此我们进行拷贝对象时, 字符串被显示拷贝*重新创建*了一个字符串, 而列表只是复制引用, 所以改变列表的元素会影响所有引用对象.

> 跟多关于python mutable vs immutable data type的介绍请看[这里](https://docs.python.org/2/reference/datamodel.html)

    In [1]: obj = ['name',['age',18]]

    In [2]: a=obj[:]

    In [3]: b=list(obj)

    In [4]: for x in obj,a,b:
       ...:     print id(x[0]),id(x[1])
       ...:
    4540997232 4550297144
    4540997232 4550297144
    4540997232 4550297144

    In [5]: a[0] = 'lisi'

    In [6]: b[0] = 'zhangsan'

    In [7]: for x in obj,a,b:
       ...:     print id(x[0]),id(x[1])
    ...:
    4540997232 4550297144
    4550273760 4550297144
    4550274624 4550297144

    In [8]: a[1][1] = 23

    In [9]: b[1][1] = 25

    In [10]: for x in obj,a,b:
       ....:     print id(x[0]),id(x[1])
    ....:
    4540997232 4550297144
    4550273760 4550297144
    4550274624 4550297144

    In [11]: obj
    Out[11]: ['name', ['age', 25]]

修改a b的name字段时, 由于字符串是不可改变类型的数据, 所以改变它就相当于新创建一个对象. 它的id自然会不一样, 也就不会互相影响, 但是由于list是可变类型的, 修改list里面任何字段都不会影响它的指针地址, 所以id没有变化, 也就会互相影响了. 包括原来的obj对象的年龄都变了.

那我就想复制一个对象和原来的完全没有任何影响应该怎么做呢?

这个时候深拷贝就登场了.
之前例子中出现过 `copy` 这个模块, 这个模块中除了出现过了 `copy` 方法来做浅拷贝之外, 还有一个方法 `deepcopy` , 这个方法就可以对一个对象做深拷贝.

    In [12]: import copy

    In [13]: obj = ['name',['age',18]]

    In [14]: a = copy.deepcopy(obj)

    In [15]: b = copy.deepcopy(obj)

    In [16]: a[1][1] = 23

    In [17]: b[1][1] = 25

    In [18]: a
    Out[18]: ['name', ['age', 23]]

    In [19]: b
    Out[19]: ['name', ['age', 25]]

    In [20]: obj
    Out[20]: ['name', ['age', 18]]

## 修改元组吧

前面提到了可变类型和不可变类型, 现在就来利用可变类型来修改元组.

    In [1]: t = ([], )

    In [2]: t[0] += [1]
    ---------------------------------------------------------------------------
    TypeError                                 Traceback (most recent call last)
    <ipython-input-2-91852f1a971a> in <module>()
    ----> 1 t[0] += [1]

    TypeError: 'tuple' object does not support item assignment

    In [3]: t
    Out[3]: ([1],)

    In [4]: t = ([], )

    In [5]: t[0].extend([1])

    In [6]: t
    Out[6]: ([1],)

See? 虽然它抛了异常, 但是还是修改成功了.

> += 到底在干嘛? 看[这里](http://emptysqua.re/blog/python-increment-is-weird-part-ii/)

## 可变参数作为函数参数

这个也是可变参数的坑, 直接上代码:

    In [1]: def foo(a=[]):
       ...:     a.append('yo')
       ...:     print a
       ...:

    In [2]: foo()
    ['yo']

    In [3]: foo()
    ['yo', 'yo']

    In [4]: foo()
    ['yo', 'yo', 'yo']

所以不要直接在函数参数上给可变类型赋默认值:

    In [1]: def foo(a=None):
       ...:     if not a:
       ...:         a = []
       ...:     a.append('yo')
       ...:     print a
       ...:

    In [2]: foo()
    ['yo']

    In [3]: foo()
    ['yo']

## 循环中修改列表的索引值

先看一个例子:

    In [1]: a = [2, 4, 5, 6]

    In [2]: for i in a:
       ...:     if not i % 2:
       ...:         a.remove(i)
       ...:

    In [3]: a
    Out[3]: [4, 5]

这个结果显然错了, 我们希望得到的是奇数. 这个原因就是在遍历列表的时候修改了它的索引.

再看下面这个例子:

    In [1]: a = [2, 4, 5, 6]

    In [2]: for count, i in enumerate(a):
       ...:     print count, i
       ...:     if not i % 2:
       ...:         a.remove(i)
       ...:
    0 2
    1 5
    2 6

第一个2被删除没有问题, 由于第一个2被删了, 5的索引值变成了1, 所以4就被跳过了, 导致最终结果错误.

> 最终希望看更多python高级编程技(大)巧(坑)的请戳[这里](http://dongweiming.github.io/Expert-Python/#1)
