# 哨兵对象模式

原文：[The Sentinel Object Pattern](https://python-patterns.guide/python/sentinel-object/)

哨兵对象模式是一种标准的 Pythonic 方法，在标准库和其他地方都有使用。该模式最常使用 Python 内置的 `None` 对象，但是在 `None` 可能是一个有用的值的情况下，可以使用一个独特的哨兵 `object()` 来表示缺失或未指定的数据。

在数值总是被指定的问题域中，编程是最容易的：数据库中的每个人都保证有一个名字，我们知道每个雇员的年龄，在实验的每一秒钟都有一个数据被成功收集。

但是世界上很少有那么简单的事情，所以需要有模式来处理那些对象属性或整个对象丢失的情况。有什么简单的机制可以区分有用的数据和表明数据不存在的占位符？

## Sentinel Value

传统的 Sentinel Value 模式对于 Python 程序员来说，可以从 `str.find()` 方法中熟悉。虽然它的替代方法`str.index()`更加严格，如果找不到你要找的子串就会引发一个异常，但是`find()`可以让你跳过为丢失的子串写一个异常处理程序，当子串没有被找到时返回哨兵值`-1`。这通常可以节省一行代码和一点缩进。

```python
try:
    i = a.index(b)
except:
    return

# versus
i = a.find(b)
if i == -1:
    return
```

这是一个典型的哨兵值的例子。值`-1`只是一个整数，就像函数的其他可能的返回值一样，但有一个提前商定的特殊含义--如果程序被返回`-1`，忘了检查，并试图用它作为字符串的索引，那就糟糕了！结果将不是程序员想要的。其结果将不是程序员所期望的。

如果今天发明了`str.find()`，它就会使用我们将在下面描述的Sentinel Object模式，简单地返回`None`表示 "未找到"。这样就不会出现返回值被意外地用作索引的可能性。

哨兵值在今天的Go语言中特别常见，它的设计鼓励一种总是返回字符串而不是仅仅引用字符串的编程风格--迫使程序员选择一些特定的字符串，如空字符串或一个特殊的独特的哨兵，以表示没有收集到数据或存在数据。

在Python中，Django框架因与几十年来的数据库实践相矛盾而著名，它建议你 ["避免在基于字符串的字段上使用null"](https://docs.djangoproject.com/en/dev/ref/models/fields/#null)--其经常出现的结果是，与Go一样，代码变得更简单；对空字符串的检查取代了对`None`的检查；当后来的代码试图对被证明为`None`的字符串方法进行调用时，程序不再崩溃了。

## The Null Pointer Pattern

空指针模式在Python中是不可能的，但值得一提的是，它概述了Python与那些用`nil`或`NULL`指针使其数据模型复杂化的语言之间的区别。

Python 中的每个名字要么不存在，要么存在并指向一个对象。你可以用`del name`删除一个名字，否则你可以给它分配一个新的对象；Python没有提供其他的选择。在幕后，Python 中的每个名字都是一个指针，存储着它当前指向的对象的地址。即使名字指向一个像 `None` 。（译者注：删除对象应该使用 `del`， 而不是指定为None）

这个保证支持Python的默认C语言实现中的一个有趣的哨兵模式。C语言称之为 "指针(pointer)"--将总是持有一个有效对象的地址。利用这种灵活性，C 语言程序员使用一个零的地址来表示 "这个指针目前没有指向任何东西" - 这使得零，或者许多 C 语言程序定义的 `NULL`，成为一个哨兵值。可能是 `NULL` 的指针在使用前需要被检查，否则程序会因分段故障(`segmentation  fault`)而死亡。

所有的Python值，甚至是`None`和`False`，都是具有非零地址的真实对象，这一事实意味着在C语言中实现的Python函数的值`NULL`有特殊含义：一个`NULL`指针意味着 "这个函数没有完成并返回一个值；相反，它引发了一个异常"。这使得Python下面的C代码可以避免Go代码中普遍存在的两值返回模式。

```python
# Go needs to separately represent “the return value”
# and “did this die with an error.”

byte_count, err := fmt.Print("Hello, world!")
if err != nil {
        ...
}
```

相反，调用 Python 的 C 语言例程可以只使用 C 函数支持的单一返回值来区分合法的返回值和异常。

```python
# The pointer to a Python object instead means
# “an exception was raised” if its value is NULL.

PyObject *my_repr = PyObject_Repr(obj);
if (my_repr == NULL) {
     ...
}
```
异常本身被存储在其他地方，可以使用Python C的API来检索。

## The Null Object Pattern

"空对象 "是真实有效的对象，碰巧代表一个空白值或一个不存在的项目。我是在阅读Martin Fowler写的《重构》([Refactoring by Martin Fowler](https://python-patterns.guide/fowler-refactoring/)) 一书时注意到这个模式的，这本书的解释归功于Bobby Woolf。请注意，这种模式与上一节中解释的 "空指针 "没有任何关系！它描述了一种特殊的发送方式。相反，它描述了一种特殊的哨兵对象。

想象一下一个`Employee`对象的序列，它通常有另一个雇员作为其`manager`属性，但并不总是如此。代表 "no manager "的默认Pythonic方法是将 `None` 赋值给该属性。

一个负责显示雇员资料的程序在试图调用经理的任何方法之前，将不得不检查哨兵对象`None`。

```python
for e in employees:
    if e.manager is None:
        m = 'no one'
    else:
        m = e.manager.display_name()
    print(e.name, '-', m)
```

而这种模式需要在所有接触到管理器属性的代码中重复进行。

Woolf提供了一个有趣的选择，即用一个专门用来表示 "没有人 "这个概念的对象来替换所有的 `None` 经理值。

```python
NO_MANAGER = Person(name='no acting manager')
```

雇员对象现在将被分配这个`NO_MANAGER`对象，而不是`None`，接触雇员经理的两种代码都将受益。

- 产生简单显示或总结的代码可以简单地打印或统计`NO_MANAGER` 经理对象，就像它是一个正常的雇员对象一样。当代码可以针对Null对象成功运行时，对特殊`if`语句的需求就消失了。
- 那些需要特别处理没有代理经理的雇员情况的代码现在变得更有可读性。**与其使用通用的 `is None`，不如使用特殊的 `is NO_MANAGER` 来执行检查，从而获得更多的可读性。**

虽然不是在所有情况下都合适--例如，设计Null对象以保持平均数和其他统计数据的有效性是很困难的--但Null对象甚至出现在Python标准库中：`logging`模块有一个`NullHandler`，它可以替代其他处理程序，但不做实际日志。

## Sentinel Objects

最后我们来看看哨兵对象模式本身。

标准的 Python 哨兵是内置的 None 对象，在需要提供整数、浮点数、字符串或其它有意义的值的地方使用。对于大多数程序来说，它是完全足够的，而且它的存在可以通过以下方式进行无误的测试。

```python
if other_object is None:
    ...
```

但是有两种有趣的情况，程序需要一个`None`的替代品。

首先，如果用户自己可能会尝试存储`None` 对象，那么一个通用的数据存储就不能选择使用`None` 来存储丢失的数据。

作为一个例子，Python 标准库的 `functools.lru_cache()` 在内部使用了 Sentinel Object 模式。隐藏在闭包内部的是一个完全唯一的对象，它为每个单独的缓存实例分别创建。

```python
sentinel = object()  # unique object used to signal cache misses
```

通过提供这个哨兵对象作为`dict.get()`的第二个参数--这里别名为`cache_get`，是Prebound Methods模式的一个闭合级私有例子--缓存可以区分一个其结果已经被缓存且恰好为`None`的函数调用和一个尚未被缓存的函数调用。

```python
result = cache_get(key, sentinel)
if result is not sentinel:
    ...
```

这种模式在标准库中出现过几次。

- 如上所示，`functools.lru_cache()`内部使用了一个sentinel对象。
- `bz2`模块有一个全局的`_sentinel`对象。
- `configparser`模块有一个同样定义为模块全局的 sentinal `_UNSET`。

第二个需要哨兵的有趣情况是当一个函数或方法想知道调用者是否提供了一个可选的关键字参数。通常 Python 程序员给这种参数的默认值是 `None`。但是如果你的代码真的需要知道其中的差别，那么一个哨兵对象将允许你检测它。

Fredrik Lundh的 [“Default Parameter Values in Python”](http://effbot.org/zone/default-values.htm) 是对使用参数默认值的早期描述，多年来，Ian Bicking的 [“The Magic Sentinel”](http://www.ianbicking.org/blog/2008/12/the-magic-sentinel.html) 和Flavio Curella的 [“Sentinel values in Python”](https://www.revsys.com/tidbits/sentinel-values-python/) 都对他们的哨兵对象缺乏可读的 `repr()` 感到担忧，并提出了各种修复方案。

但不管是什么应用，哨兵对象模式的核心是，是对象的身份--而不是它的值--让周围的代码认识到它的意义。如果你使用平等运算符来检测哨兵，那么你只是在使用本页面顶部描述的哨兵值模式。哨兵对象是通过使用 Python `is` 操作符来检测哨兵是否存在而定义的。


