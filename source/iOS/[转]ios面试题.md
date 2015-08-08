---
title: "[转]iOS面试题"
date: 2015-08-07 16:52
---

最近看到一个项目，是别人总结的iOS面试题的解法，这个项目目前还没有写完，我觉得写得挺好的，就把我感兴趣的题搬运过来。往后我会逐步加入别的题目，争取把这篇文章做成面试题的wiki。

> 原项目地址：[戳我](https://github.com/ChenYilong/iOSInterviewQuestions)

# @property相关

## 什么情况使用weak关键字，相比assign有什么不同

首先要明白 `weak` 是干什么。

`weak` 此特质表明该属性定义了一种“非拥有关系” (nonowning relationship)。为这种属性设置新值时，设置方法既不保留新值，也不释放旧值。此特质同 `assign` 类似， 然而在属性所指的对象遭到摧毁时，属性值也会清空(nil out)，也就是说 `weak` 当对象销毁的时候,指针会被自动设置为nil。 而 `assign` 的“设置方法”只会执行针对“纯量类型” (scalar type，例如 CGFloat 或 NSlnteger 等)的简单赋值操作。 `assign` 可以用非OC对象，而 `weak` 必须用于OC对象。

那么什么情况下，应该用weak关键字呢？

* 在ARC中，出现循环引用的时候，必须要有一端使用weak，比如:自定义View的代理属性(delegate)
    * 自身已经对它进行一次强引用，没有必要再强引用一次，此时也会使用 `weak`

## 什么情况下用copy关键字

    * NSString、NSArray、NSDictionary 等等经常使用copy关键字，是因为他们有对应的可变类型：NSMutableString、NSMutableArray、NSMutableDictionary(具体为什么下面的问题会提到)
    * `block` 也经常使用copy关键字， `block` 使用 `copy` 是从MRC遗留下来的“传统”，在MRC中，方法内部的 `block` 是在栈区的，使用 `copy` 可以把它放到堆区。在ARC中写不写都行：对于 `block` 使用 `copy` 还是 `strong` 效果是一样的，但写上 `copy` 也无伤大雅，还能时刻提醒我们：编译器自动对block进行了copy操作。

## 这个写法会出什么问题： `@property (copy) NSMutableArray *array;`

    * 添加，删除，修改数组内的元素的时候，程序会因为找不到对应的方法而崩溃，因为 `copy` 就是复制一个不可变NSArray的对象
    * 使用了atomic属性会严重影响性能(默认就是atomic)。该属性使用了同步锁，会在创建时生成一些额外的代码用于帮助编写多线程程序，这会带来性能问题，通过声明nonatomic可以节省这些虽然很小但是不必要额外开销。

## 什么是“自动合成”（autosynthesis）

    完成属性定义后，编译器会自动编写访问这些属性所需的方法，此过程叫做“自动合成”( autosynthesis)。需要强调的是，这个过程由编译 器在编译期执行，所以编辑器里看不到这些“合成方法”(synthesized method)的源代码。除了生成方法代码 getter、setter 之外，编译器还要自动向类中添加适当类型的实例变量，并且在属性名前面加下划线，以此作为实例变量的名字。可以在类的实现代码里通过 @synthesize语法来指定实例变量的名字（默认是加上下划线）。

## @synthesize和@dynamic分别有什么作用

    * @property有两个对应的词，一个是@synthesize，一个是@dynamic。如果@synthesize和@dynamic都没写，那么默认的就是@syntheszie var = _var;
    * @synthesize的语义是如果你没有手动实现setter方法和getter方法，那么编译器会自动为你加上这两个方法。
    * @dynamic告诉编译器：属性的setter与getter方法由用户自己实现，不自动生成。（当然对于readonly的属性只需提供getter即可）。假如一个属性被声明为@dynamic var，然后你没有提供@setter方法和@getter方法，编译的时候没问题，但是当程序运行到instance.var = someVar，由于缺setter方法会导致程序崩溃；或者当运行到 someVar = var时，由于缺getter方法同样会导致崩溃。编译时没问题，运行时才执行相应的方法，这就是所谓的动态绑定。

## 用@property声明的NSString（或NSArray，NSDictionary）经常使用copy关键字，为什么？如果改用strong关键字，可能造成什么问题

    首先要明白， `strong` 和 `copy` 它们的区别是什么

    * 因为父类指针可以指向子类对象,使用copy的目的是为了让本对象的属性不受外界影响,使用copy无论给我传入是一个可变对象还是不可对象,我本身持有的就是一个不可变的副本
    * 如果我们使用是strong,那么这个属性就有可能指向一个可变对象,如果这个可变对象在外部被修改了,那么会影响该属性

    copy此特质所表达的所属关系与strong类似。然而设置方法并不保留新值，而是将其“拷贝” (copy)。 当属性类型为NSString时，经常用此特质来保护其封装性，因为传递给设置方法的新值有可能指向一个NSMutableString类的实例。这个类是NSString的子类，表示一种可修改其值的字符串，此时若是不拷贝字符串，那么设置完属性之后，字符串的值就可能会在对象不知情的情况下遭人更改。所以，这时就要拷贝一份“不可变” (immutable)的字符串，确保对象中的字符串值不会无意间变动。只要实现属性所用的对象是“可变的” (mutable)，就应该在设置新属性值时拷贝一份。

    在非集合类对象中：对immutable对象进行copy操作，是指针复制，mutableCopy操作时内容复制；对mutable对象进行copy和mutableCopy都是内容复制。
