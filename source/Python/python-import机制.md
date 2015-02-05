---
title: "python import机制"
date: 2015-02-05 22:50
---

今天王颖同学问了我一个问题，大概就是说mongo的数据库连接应该写在哪里，
然后我告诉他应该写在models层的 ** __init__.py ** 文件中，这样整个项目启动的时候会初始化得到一个mongo的连接，
整个项目就能通过这个连接来完成数据库操作。

实际上，我们正式通过python的import机制，实现了一个数据库连接的单例模式。

王颖同学虽然是c++大神，但是毕竟不太懂python，所以我觉得要想真正的理解这个设计，就必须搞清楚python的import机制。
于是我问了王颖同学这样一个问题:

假设在同一个目录下有这样3个文件，分别是a.py, b.py, c.py，代码如下:

    # a.py
    import b
    print b.attr

    # b.py
    import a
    attr = "Hello world!"

    # c.py
    import a

此时 `python c.py` 会得到什么呢?

王颖同学认为此时应该会爆出循环引用的错误，实则不然，正确的结果是会输出 `Hello world!` 。

当import一个模块的时候，python会去看sys.modules里面有没有这个模块，如果有，
则跳过import语句，执行后面的内容，如果没有则会降模块放入sys.modules字典中，
并载入模块中内容，如:

    In [1]: import sys

    In [2]: 'requests' in sys.modules
    Out[2]: False

    In [3]: import requests

    In [4]: 'requests' in sys.modules
    Out[4]: True

更多的信息则放在模块的 `__dict__` 属性中。

当执行c.py的时候，执行import a这条语句：

1. 判断a是否在sys.modules里面，发现不在，则创建一个新的module对象a放到sys.modules中
然后执行a.py其他语句，填充module a的 `__dict__` 属性。
2. a.py中第一句就是import b，所以又开始载入b模块，同理，会在sys.modules中创建module对象b。
3. 然后执行b.py中其他语句，b.py中第一句就是import a，由于a模块已经在sys.modules中有了，
所以会跳过这条import，执行b.py中下一条语句，也就是 `attr = "Hello world!"` 并且放到了b模块对象
的 `__dict__` 属性中。
4. 接着会执行a.py中剩余语句，也就打印出了 `b.attr` ，也就是 "Hello world!"。

由于python不会重复import一个模块，所以也就保证了数据库连接初始化只会执行一次。
写在 ** __init__.py ** 中是保证会最先被执行到，因为 ** __init__.py ** 是表示这个文件夹是一个python的package，它会首先被执行到。
