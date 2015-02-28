---
title: "python的装饰器"
date: 2015-02-28 17:06
---

## 基础概念

Python的装饰器是在实际开发中非常好用的东西，有点类似于c里面的宏，或者java的AOP，要理解它也很容易。

首先，python中所有东西都是对象，包括函数，函数是first-class对象，所以函数可以作为函数形参，可以作为返回值，也可以直接赋值给另一个变量并且支持闭包函数。如：

    In [1]: def f_a():
      ....:     print "function a"
      ....:

    In [2]: def f_b():
      ....:     return f_a
      ....:

    In [3]: def f_c(f):
      ....:     f()
      ....:

    In [4]: f_d = f_b()

    In [5]: f_d()
    function a

    In [6]: f_c(f_d)
    function a

    In [7]: def f_e(name):
      ....:     def f_f():
      ....:         print "hello" +     str(name)
      ....:     f_f()

    In [8]: f_e("crow")
    hellocrow

这里需要注意的是，python的闭包函数比如f_f可以访问到父控件的变量。
知道这些概念之后，看一个简单的装饰器，它会打出被装饰的函数的执行时间：

    In [9]: def show_time(f):
       ...:     def _(*args, **kwargs):
       ...:         start = time.time()
       ...:         f(*args, **kwargs)
       ...:         print "%.3fs costs {%s}" % (time.time() - start, f.__name__)
       ...:     return _

    In [10]: @show_time
        ...: def wait():
        ...:     time.sleep(1)
        ...:

    In [11]: wait()
    1.005s costs {wait}

装饰器本质上就是函数赋值和闭包的结合， `@` 不过是一个语法糖而已。 `show_time` 函数以被装饰的函数作为形参传入，在函数体重定义了一个内部函数 `_` ，这个函数接受任意的参数，调用传进来的函数，然后计时。

但是这样有一个问题，被装饰的wait函数，它的一部分属性被替换成了 `_` 函数的，比如:

    In [12]: wait.__name__
    Out[12]: '_'

解决这个问题的方法很多，最优雅的方式就是使用python自带的functools模块的wraps装饰器：

    In [13]: from functools import wraps

    In [14]: def show_time(f):
       ....:     @wraps(f)
       ....:     def _(*args, **kwargs):
       ....:         start = time.time()
       ....:         f(*args, **kwargs)
       ....:         print "%.3fs costs {%s}" % (time.time() - start, f.__name__)
       ....:     return _
       ....:

    In [15]: @show_time
       ....: def wait():
       ....:     time.sleep(1)
       ....:

    In [16]: wait()
    1.005s costs {wait}

    In [17]: wait.__name__
    Out[17]: 'wait'

## 带参数的装饰器

这个也没啥特殊的，本质上来讲，就是在利用闭包，子函数能够访问上层空间的属性这个特点，再包一层，最外层作用仅仅是为了让里层的函数能够访问装饰器的参数，里面还是和普通的装饰器一样。比如：

    In [18]: def show_time(name="crow"):
       ....:     def wrapper(f):
       ....:         @wraps(f)
       ....:         def _(*args, **kwargs):
       ....:             print "hello " + str(name)
       ....:             start = time.time()
       ....:             f(*args, **kwargs)
       ....:             print "%.3fs costs {%s}" % (time.time() - start, f.__name__)
       ....:         return _
       ....:     return wrapper
       ....:

    In [19]: @show_time("world")
       ....: def wait():
       ....:     time.sleep(1)
       ....:

    In [20]: wait()
    hello world
    1.004s costs {wait}

这里实际上发生的是:

    _ = show_time(name)
    wait = _(wait)

## 类作为装饰器

类当然也可以作为装饰器，只要类实现了 `__call__` 方法。
比如我们首先重写不带参数的show_time装饰器：

    In [1]: import time

    In [2]: class show_time(object):
       ...:     def __init__(self, f):
       ...:         self.f = f
       ...:     def __call__(self, *args, **kwargs):
       ...:         start = time.time()
       ...:         self.f(*args, **kwargs)
       ...:         print "%.3fs costs {%s}" % (time.time() - start, self.f.__name__)
       ...:

    In [3]: @show_time
       ...: def wait():
       ...:     time.sleep(1)
       ...:

    In [4]: wait()
    1.003s costs {wait}

实际上 `@` 的语法糖被翻译成：

    wait = show_time(wait)
    wait()

这里首先创建了一个show_time的对象 `wait` ，然后 `wait()` 的时候调用了wait对象的 `__call__` 方法。

再看一下带参数的写法：

    In [5]: class show_time(object):
       ...:     def __init__(self, *args, **kwargs):
       ...:         print args[0]
       ...:     def __call__(self, f):
       ...:         def _(*args, **kwargs):
       ...:             start = time.time()
       ...:             f(*args, **kwargs)
       ...:             print "%.3fs costs {%s}" % (time.time() - start, f.__name__)
       ...:         return _
       ...:

    In [6]: @show_time('crow')
       ...: def wait():
       ...:     time.sleep(1)
       ...:
    crow

    In [7]: wait()
    1.004s costs {wait}

这里被翻译成：

    _ = show_time(*args, **kwargs)
    wait = _(wait)
    wait()

首先创建了一个show_time对象 `_` ，此时把装饰器的参数传到对象中，然后调用 `_(wait)` 的时候调用 `_` 对象的 `__call__` 方法，返回了一个函数，这个函数接受被装饰的函数，并返回调用时间。

## 一些好用的装饰器

python中的方法，一般都是叫做 **bound method** ，调用这些 **bound method**
 的时候，会把调用方法的对象作为第一个参数传进去，这很自然，很多语言都是这样，比如js的this。

    In [1]: class A(object):
       ...:     def m1(self, *args):
       ...:         return args
       ...:     def m2(*args):
       ...:         return args

    In [2]: a.m1
    Out[2]: <bound method A.m1 of <__main__.A object at 0x104857c50>>

    In [3]: a.m1()
    Out[3]: ()

    In [4]: a.m2('hello')
    Out[4]: (<__main__.A at 0x104857c50>, 'hello')

手动先写self，这样能避免取到对象的指针而引起错误。

### classmethod

如果方法被这个装饰器装饰，这个方法将还是bound method，只不过第一个参数不是对象，而是类。并且可以通过类名直接访问该方法。

    In [5]: class A(object):
       ....:     @classmethod
       ....:     def m(*args):
       ....:         return args
       ....:

    In [6]: a = A()

    In [7]: a.m
    Out[7]: <bound method type.m of <class '__main__.A'>>

    In [8]: a.m("hello")
    Out[8]: (__main__.A, 'hello')

    In [9]: A.m("hello")
    Out[9]: (__main__.A, 'hello')

### staticmethod

被这个装饰器装饰的方法将使普通的方法，不会有任何对象被隐式的传递。同样也能通过类或者对象访问。

    In [10]: class A(object):
       ....:     @staticmethod
       ....:     def m(*args):
       ....:         return args
       ....:

    In [11]: a = A()

    In [12]: a.m
    Out[12]: <function __main__.m>

    In [13]: a.m()
    Out[13]: ()

    In [14]: A.m()
    Out[14]: ()

## 总结

写装饰器或者看装饰器的时候，如果不带参数的装饰器（@后面的装饰器名后面没有括号）就想成：

    f = wrapper(f)

有参数（有括号就先接形参）就想成：

    _ = wrapper(*args, **kwargs)
    f = _(f)

这样就方便理解了。

> 更多请看[PythonDecoratorLibrary](https://wiki.python.org/moin/PythonDecoratorLibrary)
