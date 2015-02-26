---
title: "python中的下划线"
date: 2015-02-26 18:36
---

python项目中会有大量的以下划线开头的属性，这些下划线开头的属性，开上去很奇怪，出现频率也很多，所以下面我总结了常见下划线的意义。

## 单下划线开头(_)

1. `_` 在代码中单独出现，这种情况 `_` 表示一个零时变量。
这样别人看代码的时候，就知道这是一个没有意义的零时变量，比起叫tmp或者temp更加简洁。

        for _ in range(10):
            do_something_ten_times()

2. 以 _ 开头的变量，如 `_foo` ，这样则表示或者约定该变量为私有变量。比如直接 `from <module> import *` 就不会import单下划线开头的属性（除非手动在 `__all__` 属性中声明）。

3. `_` 在解释器中出现，则表示上一个执行的结果，如：

        In [1]: 1 + 1
        Out[1]: 2

        In [2]: _
        Out[2]: 2

## 双下划线开头(__)

1. 双下划线开头以及结尾( `__init__` )，这些一般都是python自带的一些magic method。之所以被双下划线包起来，是为了区别用户自己定义方法。
2. 最少2个下划线开头，最多一个下划线结尾，python会对这些方法名加入类名后重组，如

        In [1]: class A(object):
        ...:     a = 1
        ...:     __b = 2
        ...:     def __f(self):
        ...:         pass
        ...:

        In [2]: a = A()

        In [3]: dir(a)
        Out[3]:
        ['_A__b',
         '_A__f',
         '__class__',
         '__delattr__',
         '__dict__',
         '__doc__',
         '__format__',
         '__getattribute__',
         '__hash__',
         '__init__',
         '__module__',
         '__new__',
         '__reduce__',
         '__reduce_ex__',
         '__repr__',
         '__setattr__',
         '__sizeof__',
         '__str__',
         '__subclasshook__',
         '__weakref__',
         'a']

        In [4]: getattr(a, 'a')
        Out[4]: 1

        In [5]: getattr(a, '__b')
        ---------------------------------------------------------------------------
        AttributeError                            Traceback (most recent call last)
        <ipython-input-5-ae11a9e10a99> in <module>()
        ----> 1 getattr(a, '__b')

        AttributeError: 'A' object has no attribute '__b'

        In [6]: getattr(a, '_A__b')
        Out[6]: 2

    a中的仅以双下划线开头的属性都被替换成了 **_CLASSNAME__XX** 的形式。这样就像java里面的final关键字一样，让这些属性不会被子类所重写。
