# 预绑定方法模式

原文：[The Prebound Method Pattern](https://python-patterns.guide/python/prebound-methods/)

在某些情况下，一个 Python 模块希望在模块的全局命名空间中提供几个例程，这些例程在运行时需要彼此共享状态。

最著名的例子可能是 Python 标准库的 `random` 模块。虽然它确实为高级用户提供了建立他们自己的`random` 实例的选项，但大多数程序员选择了在模块的全局命名空间中提供的方便的例程--像 `randrange()`、`randint()`和 `choice() `--它们反映了一个随机对象的方法。

这些顶级例程是如何共享状态的？

在幕后，这些可调用的方法，事实上，已经提前绑定到一个单一的 `Random` 实例，该模块本身已经提前构建。

在回顾了预绑定方法模式所解决的问题之后，我们将看到该模式在Python代码中的样子。

## Alternatives

在一对模块级的可调用程序之间共享状态的最原始的方法是编写一对函数，操作存储在它们旁边的模块顶层的全局数据。

想象一下，我们想提供一个简单的随机数发生器。它在一个无尽的循环中，以一个固定的伪随机顺序返回数字1到255。我们还想提供一个简单的 `set_seed()` 例程，将生成器的状态重设为一个已知的值--这对于使用随机数的测试和希望提供可重复结果的伪随机性驱动的模拟都很重要。

如果Python只支持普通函数，我们可以将共享种子存储为一个模块全局，我们的函数可以直接访问和修改。

```python
from datetime import datetime

_seed = datetime.now().microsecond % 255 + 1

def set_seed(value):
    global _seed
    _seed = value

def random():
    global _seed
    _seed, carry = divmod(_seed, 2)
    if carry:
        _seed ^= 0xb8
    return _seed
```

这种方法有几个问题。

首先，不可能再实例化出这个随机数生成器的第二个副本。如果两个线程都想拥有自己的生成器，以避免用锁来保护它，那么他们就会很不走运。

(好吧，不是真的；这是 Python。想想各种可能性吧! 你可以导入模块，保存对它的引用，从 `sys.modules` 字典中删除它，然后再次导入它以获得第二个副本。或者你可以手动实例化第二个模块对象，然后把三个名字都复制过来。我所说的 "不走运 "只是指正常的Pythonic方法）。

第二，如果随机数生成器的状态是全局的，那么将你的随机数生成器的测试从彼此之间解耦就更加困难。每个测试都要创建和行使一个单独的隔离实例，而不是每个测试都要共享一个发生器，并在下一个测试运行前正确地重置其状态。

第三，这种方法放弃了封装。这听起来比前两个抱怨更模糊，但对于一个由两个函数和一个`seed` 组成的小型紧密系统来说，它可能会降低可读性（"可读性很重要"），在一个可能包含几十个其他对象的模块中，它被分散成三个独立的、不明显相关的名字。

为了解决上述问题，我们缩进了这两个函数，并把它们包装成一个新的 Python 类的方法。然后这两个方法和它们的状态就可以愉快地捆绑在一起。

```python
from datetime import datetime

class Random8(object):
    def __init__(self):
        self.set_seed(datetime.now().microsecond % 255 + 1)

    def set_seed(self, value):
        self.seed = value

    def random(self):
        self.seed, carry = divmod(self.seed, 2)
        if carry:
            self.seed ^= 0xb8
        return self.seed
```
这给调用者带来了一个额外的步骤：在调用这两个方法之前，对象必须被实例化。有什么办法可以让调用者跳过这一步吗？

软件工程师最好先停下来，认真思考一下，在绝大多数情况下，只要定义一个这样的类就足够了。你的用户只需要一条语句就可以创建一个你的类的实例。如果他们自己的应用程序的架构被深深地破坏了，不能轻易地把实例传递到需要的地方，他们总是可以把实例作为模块全局存储在他们自己的一个模块中，供其他代码使用。

有几个原因可以说明Prebound Methods模式特别适合于随机数发生器。

1. 实例化一个随机数生成器需要一个系统调用--在我们的例子中，需要询问日期；对于Python `random` 模块，需要从系统熵池中获取字节。如果每个需要随机数的模块都必须实例化它自己的 `Random` 对象，那么这个成本就会反复产生。
2. 伪随机数生成器是一个有趣的例子，它的行为在共享时可以更加理想。如果你是一个实例的唯一调用者，你会看到其完全可预测的重复值序列。如果你与其他代码共享该实例，那么每次其他调用者自己调用生成器时，生成器都会在其序列中不可预测地向前跳过。
3. 由于大多数`random`的用户，包括标准库中的几个模块，专门导入它以使用它的模块级调用，所以很少有预建的`Random` 实例为它们提供动力而闲置不用的情况。

如果你正在设计的自己的模块的成本和收益达到了类似的平衡，那么预置方法模式可以让你提供显著的便利。

## The pattern

为了给你的用户提供一板一眼的预制方法。

- 在模块的最高层实例化你的类。
- 考虑给它分配一个前缀为下划线_的私有名称，不要让用户直接插手这个对象。
- 最后，将该对象的每个方法的绑定副本分配给模块的全局命名空间。

对于我们上面用作说明的随机数发生器，整个模块可能看起来像这样。

```python
from datetime import datetime

class Random8(object):
    def __init__(self):
        self.set_seed(datetime.now().microsecond % 255 + 1)

    def set_seed(self, value):
        self.seed = value

    def random(self):
        self.seed, carry = divmod(self.seed, 2)
        if carry:
            self.seed ^= 0xb8
        return self.seed

_instance = Random8()

random = _instance.random
set_seed = _instance.set_seed
```

现在，用户将能够调用每个方法，就像它是一个独立的函数。但是这些方法将秘密地共享状态，这要归功于它们所绑定的公共实例，而不需要用户明确地管理或传递这些状态。

当行使这种模式时，请对在导入时实例化对象的危险性负责。这种模式通常不适合那些构造函数创建文件、读取数据库配置、打开套接字，或者一般来说会对导入它们的程序造成副作用的类。在这种情况下，你最好完全避免使用预置方法模式，或者将任何实际的副作用推迟到其中一个方法被调用。你可以选择一个中间地带，即提供一个 `setup()` 方法，并要求程序员在期待其他例程工作之前调用它。

但是对于那些可以被实例化的轻量级对象来说，Prebound Methods模式是一种优雅的方式，可以使类实例的有状态行为在模块的全局水平上可用。

这种模式的例子在标准库中随处可见，甚至超出了我们用来作为例子的`random` 模块。`calendar.py` 模块使用了这种模式。

```python
c = TextCalendar()

...

monthcalendar = c.monthdayscalendar
prweek = c.prweek
week = c.formatweek
weekheader = c.formatweekheader
prmonth = c.prmonth
month = c.formatmonth
calendar = c.formatyear
prcal = c.pryear
```

就像古老的`reprlib.py`一样，提供一个单一的`repr()`例程。
```python
aRepr = Repr()
repr = aRepr.repr
```

和其他几个模块一样--你会发现这些标准库的每个对象都使用了 Prebound Methods 模式。

- `distutils.log._global_log`
- `multiprocessing.forkserver._forkserver`
- `multiprocessing.semaphore_tracker._semaphore_tracker`
- `secrets._sysrand`

最后一个提示：将方法明确分配给全局名称几乎总是更好。即使有一打方法，我也建议继续写一打赋值语句的快速堆栈，这些赋值语句排列在你模块的左手边，使你正在定义的整个全局名称列表可见。Python 是一种动态语言，这一事实可能会诱使你使用属性自省和 `for `循环来自动完成一系列的赋值。我建议不要这样做。Python 程序员相信 "显式比隐式好"--将名字的堆栈具体化为真正的代码将更好地支持人类读者、语言服务器，甚至是古老的 `grep`。
