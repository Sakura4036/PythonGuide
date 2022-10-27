# Global Object Pattern

## Verdict

像其他几种脚本语言一样，Python 将每个模块的外层解析为普通代码。未加边框的赋值语句、表达式、甚至循环和条件语句都将在模块被导入时执行。这为用常量和数据结构来补充模块的类和函数提供了一个很好的机会，调用者会发现它很有用--但也提供了危险的诱惑：可变的全局对象会把远处的代码耦合起来，而 I/O 操作会带来导入时间的费用和副作用。

每个 Python 模块都是一个独立的命名空间。像 `json` 这样的模块可以提供一个 `loads()` 函数，而不会与 `pickle` 模块中定义的完全不同的 `loads()` 函数发生冲突、替换或覆盖。

独立的命名空间对于使编程语言具有可操作性至关重要。如果 Python 模块不是独立的名字空间，你将无法通过把注意力集中在你面前的模块上来阅读或编写 Python 代码 -- 一行代码可能会使用，或意外地与标准库中其他地方定义的名字或你安装的第三方模块冲突。如果一个第三方模块的新版本定义了一个新的全局，并与你的模块发生冲突，那么升级该模块可能会破坏你的整个程序。被迫在没有命名空间的语言中编码的程序员很快就会发现自己在全局名称中添加了前缀、后缀和额外的标点符号，以防止它们发生冲突。

当然，每个函数和类都是一个对象--在 Python 中，**所有东西都是一个对象**——模块全局模式更具体地指的是在模块的全局级别上被赋予名字的普通对象实例。

有两种模式使用了模块全局，但它们的重要性足以让它们有自己的文章。

- [Prebound Methods](https://python-patterns.guide/python/prebound-methods/) 是在一个模块构建一个对象时产生的，然后将该对象的一个或多个绑定方法分配给模块全局级别的名称。这些名字可以在以后用来调用这些方法，而不需要去找对象本身。
- 虽然一个 [Sentinel Object](https://python-patterns.guide/python/sentinel-object/) 不一定要生活在模块的全局命名空间中--一些哨兵对象被定义为类的属性，而另一些则是私有的，生活在一个闭包中--但许多哨兵，无论是在标准库中还是在其他地方，都被定义为模块全局并被访问。

本文将介绍其他一些常见的情况。

## Constant Pattern

模块经常将有用的数字、字符串和其他值分配给其全局范围内的名字。标准库包括许多这样的赋值，我们可以从中摘录几个例子。

```python
January = 1                   # calendar.py
WARNING = 30                  # logging.py
MAX_INTERPOLATION_DEPTH = 10  # configparser.py
SSL_HANDSHAKE_TIMEOUT = 60.0  # asyncio.constants.py
TICK = "'"                    # email.utils.py
CRLF = "\r\n"                 # smtplib.py
```
记住，这些是 "常量"，只是在对象本身是不可改变的意义上。名称仍然可以被重新分配。

```python
import calendar
calendar.January = 13
print(calendar.January) # 13
```
或被删除，对于这个问题。
```python
del calendar.January
print(calendar.January)
```
```shell
Traceback (most recent call last):
  ...
AttributeError: module 'calendar' has no attribute 'January'
```

除了整数、浮点数和字符串之外，常量还包括不可变的容器，如 `tuples` 和 `frozen sets`。
```python
all_errors = (Error, OSError, EOFError)  # ftplib.py
bytes_types = (bytes, bytearray)         # pickle.py
DIGITS = frozenset("0123456789")         # sre_parse.py
```

更专门的不可变数据类型也可以作为常量。
```python
_EPOCH = datetime(1970, 1, 1, tzinfo=timezone.utc)  # datetime
```

在极少数情况下，代码中明显不打算修改的模块全局还是会使用可变数据结构。在 `frozenset` 发明之前的代码中，普通的可变集很常见。今天仍在使用字典，因为，不幸的是，标准库没有提供冻结字典。
```python
# socket.py
_blocking_errnos = { EAGAIN, EWOULDBLOCK }
```

```python
# locale.py
windows_locale = {
  0x0436: "af_ZA", # Afrikaans
  0x041c: "sq_AL", # Albanian
  0x0484: "gsw_FR",# Alsatian - France
  ...
  0x0435: "zu_ZA", # Zulu
}
```

常量通常是作为重构引入的：程序员注意到相同的值 `60.0`反复出现在他们的代码中，因此为该值引入了一个常量`SSL_HANDSHAKE_TIMEOUT`。现在，每次使用这个名字都会导致在全局范围内进行搜索的轻微代价，但这被一些优势所平衡。常量的名称现在记录了值的含义，提高了代码的可读性。常量的赋值语句现在提供了一个单一的位置，将来可以在那里编辑这个值，而不需要在代码中寻找每个使用 `60.0` 的地方。

这些优点是非常重要的，有时甚至会为一个只使用一次的值引入常量，将一个隐藏在代码深处的字词作为一个全局值提升到可见的位置。

一些程序员把常量分配放在靠近使用它们的代码的地方；另一些程序员则把所有常量放在文件的顶部。除非常量被放在离代码很近的地方，以至于人类读者总是能看到它，否则把常量放在模块的顶部会更友好，便于那些还没有把编辑器配置为支持跳转定义的读者参考。

另一种常量不是向内的，针对模块本身的代码，而是向外的，作为模块所宣传的 API 的一部分。像 `logging` 模块中的`WARNING` 这样的常量为调用者提供了常量的优势：代码将更加可读，常量的值可以在以后调整，而不需要每个调用者编辑他们的代码。

你可能会想，一个供模块自己使用，但不供调用者使用的常量，总是以下划线开始，以标明它的私有。但是 Python 程序员在标记常量为私有时并不一致，也许是因为需要永远保留一个常量，因为调用者可能已经决定开始使用它，这比让一个辅助函数或类的 API 永远被锁定的代价要小。
