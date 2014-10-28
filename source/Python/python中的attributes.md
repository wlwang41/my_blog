---
title: "Python中的attributes"
date: 2014-10-27 22:07
---

前言：这篇文章主要是翻译整理了Pycoders's Weekly邮件推荐的一篇文章，原文地址请戳[这里](http://blog.lerner.co.il/python-attributes/)

在python中，声明一个类是非常容易的，比如：

    class Foo(object):
        def __init__(self, x, y):
            self.x = x
            self.y = y

这样就声明了一个名叫Foo的类，有这个类创造一个对象也同样简单：

    f = Foo(20, 'hello')

从表面上看，我们创建了一个Foo的对象f，并且为f的2个实例变量赋了初始值。
但是实际上在python中并没有所谓的实例变量(instance variables)和类变量(class variables)这一说，
它们在python的oop里面，都只是对象的属性(attributes)而已。
而在python的世界中，所有变量都是对象，所以实际上，属性是整个python的基础。
所有对象都有属性，通过built-in的 `dir([object]) -> list of strings` 函数可以看到对象的属性:

- 对于模块对象(module object): 返回模块的属性
- 对于类对象(class object): 返回它的属性并且递归的返回它所有父类的属性
- 对于其他的对象: 返回它的属性，它类的属性，以及各个父类的属性

所以对于python中的变量都可以通过 `dir` 函数来查看它的属性：

    In [3]: s = "hello"

    In [4]: len(dir(s))
    Out[4]: 71

    In [5]: i = 123

    In [6]: len(dir(i))
    Out[6]: 64

    In [7]: t = (1, 2, 3)

    In [8]: len(dir(t))
    Out[8]: 32

    In [9]: def f():
       ...:     pass
       ...:

    In [10]: len(dir(f))
    Out[10]: 31

可以看到，方法都有属性，因为方法也是对象嘛，这个js程序员应该不会陌生，js里面也是所有变量都是对象。
我们设置可以对方法进行赋值：

    In [11]: f.a = 1

    In [12]: f.a
    Out[12]: 1

说到对属性的赋值，python提供了2个函数来为对象的属性赋值以及取属性的值。他们是 `getattr(object, name[, default]) -> value` 以及 `setattr(object, name, value)`.

在我刚写python的时候，有一个问题当时困扰了我一会，当时我需要根据字符串的方法名来调用这个方法。这个问题可以进一步抽象为，怎么通过一个 `dir` 方法得到那些字符串为对象的"同名"属性来赋值或者取值?

这里就可以用 `getattr` 和 `setattr` 来实现，如：

    In [19]: getattr(s, "split")
    Out[19]: <function split>

这里等同于:

    In [20]: s.split
    Out[20]: <function split>

实际上，这种"."(dot notation)方式取对象的属性只是 `getattr` 函数的一个语法糖而已。
同理，`f.x = 10` 也只是 `setattr(f, "x", 10)` 的语法糖。
并且在python中，可以为对象的属性赋任何类型的值，就算赋一个复杂的对象也没有关系。
当然，如果对对象中一个没有声明过的属性赋值，会先创建这个属性。如:

    In [21]: f.new_attr = "hello"

    In [22]: f.school = "blue_shit"

虽然我们可以随时为对象增加属性，但是一个好的编程规范就是把属性的初始化操作都封装到 `__init__` 方法中，而不是到处随时初始化一个属性。

对于一个对象来说，我们可以随时对它初始化一个属性，对于类来说也是一样，如：

    In [24]: class F(object):
    ....:     pass
    ....:

    In [25]: F.age = 21

    In [26]: F.age
    Out[26]: 21

类也是对象，所以类也会有属性。同样，到处对类初始化属性是一个非常不好的习惯，对象可以把这些操作封装到 `__init__` 方法里面，对类来说，可以把这些操作封装到类的声明里面。对于在类的主体中初始化的属性，会被立即执行，但是通过 `def` 关键字声明的方法，则不会被执行，只有被调用到才会执行。如下的代码中，print函数就会被执行，name会被初始化，但是func方法主体则不会被执行。

    In [29]: class F(object):
    ....:     name = "crow"
    ....:     print "in class"
    ....:     def func():
    ....:         print "in func"
    ....:
    in class

    In [30]: F.name
    Out[30]: 'crow'

所以对于类来说，在类声明中初始化的变量，将会成为类对象的属性。


如果我们通过 `def` 关键字定义了一个函数，实际上我们是在全局定义了一个变量，但是如果我们同样在类中去定义一个函数(方法)，实际上我们只是为这个类对象添加了一个属性。
换句话说，实例的方法实际上是在类中，而不是在实例中的。当我们调用Foo的实例f的方法say时 `f.say()` , 实际上是调用了Foo的say方法，并且把f作为第一个参数传到say中。
所以说， `f.say()` 和 `Foo.say(f)` 没有本质上的区别。更多关于python方法和函数的区别会在以后的文章中写到。

那python是怎么把 `f.say()` 转到 `Foo.say(f)` 上的呢？

这个就和python中变量和属性的"寻址"有关了。

在python中，变量的查找遵循LEGB原则：Local，Enclosing，Global以及Builtin。而属性的查找则稍有不同，首先它会找属性声明的那个对象，然后是创造出对象的类，然后是随着类的继承链往上找，直到object对象。

Thus, in our case, we invoke “f.blah()”. Python looks on the instance f, and doesn’t find an attribute named “blah”. Thus, it looks on f’s class, Foo. It finds the attribute there, and performs some Python method rewriting magic, thus invoking “Foo.blah(f)”.

因此，在上面讨论到的例子中，我们调用 `f.say()` python会先查找对象f，没有找到一个名叫say的属性，因此就会去找f的类，Foo，然后找到了这个属性，python使用了方法的重写魔法，最后就调用到了 `Foo.say(f)` .

所以说，python实际上没有实例变量和类变量这一说，只有对象和属性。

但是我们应该避免一种情况，那就是类属性和实例属性重名的问题，例如：

    class Person(object):
        population = 0
        def __init__(self, first, last):
            self.first = first
            self.last = last
            self.population += 1

    p1 = Person('Reuven', 'Lerner')
    p2 = Person('foo', 'bar')

当得到p1和p2之后， `Person.population` 还是0， `p1.population` 和 `p2.population` 都是1.

这个是由 `self.population += 1` 导致的。

`self.population += 1` 等效于 `self.population = self.population + 1` ，首先右边表达式将会先计算，然后给左边表达式赋值。 python查看实例对象，self，发现没有population这个属性，然后就会去查它的Person类，结果发现了这个属性，取它的值0，加一然后重新付给了实例的population属性。所无论有多少个Person对象被创建出来，它们的population属性都会是1，类的population都是0.

最后，这篇文章只是很简单的介绍，更详细的关于python oop，函数以及方法等等都会以后慢慢给出。
