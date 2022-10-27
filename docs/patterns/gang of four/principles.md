# 基本原则

**翻译来源：[python-patterns.guide](https://python-patterns.guide/gang-of-four/composition-over-inheritance/)**

## 组合大于继承

**Favor object composition over class inheritance.**

在Python和其他编程语言中，这一伟大的原则鼓励软件架构师摆脱面向对象，转而享受基于对象编程的简单实践。

接下来通过一个设计问题，看看这个原则是如何通过几个经典的“GOF”设计模式实现的。每个设计模式都将把简单的类组装成一个优雅的运行时解决方案，不受继承的负担。

## 子类爆炸
继承作为一种设计策略的一个重要弱点是，一个类经常需要同时沿着几个不同的设计轴进行专业化，这导致了四人帮在他们的 Bridge 章节中所说的 "类的扩散 "和在 Decorator 章节中所说的 "支持各种组合的子类的爆炸"。

Python 的日志模块是标准库中一个很好的例子，它本身就是一个遵循 "组合高于继承 "原则的模块，所以我们用日志作为例子。想象一下一个基础的日志类，随着开发者需要将日志信息发送到新的目的地，它逐渐获得了子类。

```python
import sys
import syslog

# The initial class.
class Logger(object):
    def __init__(self, file):
        self.file = file

    def log(self, message):
        self.file.write(message + '\n')
        self.file.flush()

# Two more classes, that send messages elsewhere.
class SocketLogger(Logger):
    def __init__(self, sock):
        self.sock = sock

    def log(self, message):
        self.sock.sendall((message + '\n').encode('ascii'))

class SyslogLogger(Logger):
    def __init__(self, priority):
        self.priority = priority

    def log(self, message):
        syslog.syslog(self.priority, message)
```

当这第一个设计轴被另一个设计轴所连接时，问题就出现了。让我们想象一下，现在需要对日志信息进行过滤：一些用户只想看到带有 "Error "字样的信息，而一个开发者用一个新的`Logger`子类来回应。
```python
class FilteredLogger(Logger):
    def __init__(self, pattern, file):
        self.pattern = pattern
        super().__init__(file)

    def log(self, message):
        if self.pattern in message:
            super().log(message)

# It works.
f = FilteredLogger('Error', sys.stdout)
f.log('Ignored: this is not important')
f.log('Error: but you want to see this')
```
```
Error: but you want to see this
```

这个陷阱现在已经埋下了，当应用程序需要过滤消息，但要把它们写到套接字而不是文件中时，这个陷阱就会被揭开。现有的类中没有一个涵盖这种情况。如果开发者继续进行子类化，并创建一个结合了两个类特性的`FilteredSocketLogger`，那么子类爆炸就开始了。

也许程序员会很幸运，不需要进一步的组合。但在一般情况下，应用程序最终会有3×2=6个类。
```
Logger            FilteredLogger
SocketLogger      FilteredSocketLogger
SyslogLogger      FilteredSyslogLogger
```

如果m和n都继续增长，那么类的总数将呈几何级数增长。这就是 "四人帮 "想要避免的 "类的扩散(proliferation of classes) "和 "子类的爆炸(explosion of subclasses)"。

解决办法是承认一个既负责过滤消息又负责记录消息的类太复杂了。在现代面向对象的实践中，它将被指责为违反了 **"单一责任原则"**。

但是，我们怎样才能将消息过滤和消息输出这两个功能分布在不同的类中呢？

## 方案1: 适配器（Adapter）模式


一种解决方案是适配器模式：决定不需要改进原来的记录器类，因为任何输出消息的机制都可以被包装成看起来像记录器所期望的文件对象。

1. 所以我们保留了原来的 `Logger`。
2. 而且我们也保留了`FilteredLogger`。
3. 但我们不是创建特定于目的地的子类，而是使每个目的地适应文件的行为，然后将适配器传递给`Logger`作为其输出文件。

```python
import socket

class FileLikeSocket:
    def __init__(self, sock):
        self.sock = sock

    def write(self, message_and_newline):
		self.sock.sendall(message_and_newline.encode('ascii'))

    def flush(self):
        pass

class FileLikeSyslog:
    def __init__(self, priority):
        self.priority = priority

    def write(self, message_and_newline):
        message = message_and_newline.rstrip('\n')
        syslog.syslog(self.priority, message)

    def flush(self):
        pass
```

Python 鼓励`duck typing`（译者注：鸭子类型是一种编程风格，决定一个对象是否有正确的接口，关注点在于它的方法或属性，而不是它的类型——“如果它看起来像鸭子，像鸭子一样嘎嘎叫，那么它一定是鸭子。”），所以适配器的唯一责任是提供正确的方法 -- 例如，我们的适配器不需要继承它们所包裹的类或它们所模仿的文件类型。他们也没有义务重新实现一个真正的文件所提供的全部十几种方法。就像如果你只需要鸭子的叫声，那么鸭子会不会走路并不重要，我们的适配器只需要实现记录器真正使用的两个文件方法。

这样就避免了子类的爆炸! 记录器对象和适配器对象可以在运行时自由混合和匹配，而不需要创建任何进一步的类。

```python
sock1, sock2 = socket.socketpair()

fs = FileLikeSocket(sock1)
logger = FilteredLogger('Error', fs)
logger.log('Warning: message number one')
logger.log('Error: message number two')

print('The socket received: %r' % sock2.recv(512))
```
```
The socket received: b'Error: message number two\n'
```

注意，上面写出` FileLikeSocket `类只是为了举例——在现实生活中，该适配器是内置在 Python 的标准库中的。只需调用任何套接字的`makefile()`方法，就可以得到一个完整的适配器，使套接字看起来像一个文件。

## 方案2 桥接（Bridge）模式

桥接模式将一个类的行为在调用者看到的外部 "抽象 "对象和包裹在里面的 "实现 "对象之间进行分割。如果我们决定过滤属于 "抽象 "类，而输出属于 "实现 "类，那么我们就可以将桥模式应用到我们的日志例子中（也许有点武断）。

就像在Adapter案例中一样，现在有一个单独的梯队的类来管理写作。但是我们不必再去扭曲我们的输出类来匹配 Python `file` 文件对象的接口——这需要在记录器中添加一个换行，有时还需要在适配器中再次删除——我们现在可以自己定义封装类的接口。

所以让我们设计内部的 "实现 "对象来接受原始消息，而不是需要添加换行，并将接口减少到只有一个方法 `emit()`，而不是还必须支持一个通常没有用的 `flush()` 方法。

```python
# The “abstractions” that callers will see.
class Logger(object):
    def __init__(self, handler):
        self.handler = handler

    def log(self, message):
        self.handler.emit(message)

class FilteredLogger(Logger):
    def __init__(self, pattern, handler):
        self.pattern = pattern
        super().__init__(handler)

    def log(self, message):
        if self.pattern in message:
            super().log(message)

# The “implementations” hidden behind the scenes.
class FileHandler:
    def __init__(self, file):
        self.file = file

    def emit(self, message):
        self.file.write(message + '\n')
        self.file.flush()

class SocketHandler:
    def __init__(self, sock):
        self.sock = sock

    def emit(self, message):
        self.sock.sendall((message + '\n').encode('ascii'))

class SyslogHandler:
    def __init__(self, priority):
        self.priority = priority

    def emit(self, message):
        syslog.syslog(self.priority, message)
```

抽象对象和实现对象现在可以在运行时自由组合。

```python
handler = FileHandler(sys.stdout)
logger = FilteredLogger('Error', handler)

logger.log('Ignored: this will not be logged')
logger.log('Error: this is important')
```
```
Error: this is important
```

这提出了比适配器更多的对称性。文件输出是`Logger`的本机，而非文件输出则需要一个额外的类，现在，一个有效的记录器总是由一个抽象和一个实现组成。

再一次，子类的爆炸被避免了，因为两种类在运行时被组合在一起，不需要任何一个类被扩展。

## 方案3 装饰器（Decorator）模式

如果我们想对同一个日志应用两个不同的过滤器呢？上述两个解决方案都不支持多个过滤器比如说，一个是按优先级过滤，另一个是匹配一个关键词。

回顾一下上一节中定义的过滤器。我们不能堆叠两个过滤器的原因是它们提供的接口和它们包裹的接口之间存在着不对称：它们提供了一个`log()`方法，但却调用了它们的处理程序的`emit()`方法。当外部过滤器试图调用内部过滤器的`emit()`时，将一个过滤器包裹在另一个过滤器中会导致 `AttributeError`。

如果我们把我们的过滤器和处理程序转变成提供相同的接口，使它们都提供一个`log()`方法，那么我们就达到了装饰器模式。

```python
# The loggers all perform real output.
class FileLogger:
    def __init__(self, file):
        self.file = file

    def log(self, message):
        self.file.write(message + '\n')
        self.file.flush()

class SocketLogger:
    def __init__(self, sock):
        self.sock = sock

    def log(self, message):
        self.sock.sendall((message + '\n').encode('ascii'))

class SyslogLogger:
    def __init__(self, priority):
        self.priority = priority

    def log(self, message):
        syslog.syslog(self.priority, message)

# The filter calls the same method it offers.
class LogFilter:
    def __init__(self, pattern, logger):
        self.pattern = pattern
        self.logger = logger

    def log(self, message):
        if self.pattern in message:
            self.logger.log(message)
```
这是第一次，过滤代码被移到任何特定的记录器类之外。相反，它现在是一个独立的功能，可以被包裹在任何我们想要的记录器上。

就像我们的前两个解决方案一样，过滤功能可以在运行时与输出相结合，而不需要建立任何特殊的组合类。

```python
log1 = FileLogger(sys.stdout)
log2 = LogFilter('Error', log1)

log1.log('Noisy: this logger always produces output')

log2.log('Ignored: this will be filtered out')
log2.log('Error: this is important and gets printed')
```
```
Noisy: this logger always produces output
Error: this is important and gets printed
```

而且，由于Decorator类是对称的——它们提供的接口与它们包裹的接口完全相同--我们现在可以在同一个日志上堆叠几个不同的过滤器了。
```python
log3 = LogFilter('severe', log2)

log3.log('Error: this is bad, but not that bad')
log3.log('Error: this is pretty severe')
```
```
Error: this is pretty severe
```
但是请注意这个设计的对称性被打破的地方：虽然过滤器可以被堆叠，但是输出例程不能被合并或堆叠。日志信息仍然只能被写入一个输出。

## 方案4 组合大于继承

Python的日志模块想要更多的灵活性：不仅要支持多个过滤器，而且要支持单个日志信息流的多个输出。基于其他语言的日志模块的设计--主要的灵感请参见`PEP 282`的 "影响 "部分——Python的日志模块实现了自己的 _Composition Over Inheritance_ 模式。

1. 调用者与之交互的`Logger`类本身并不实现过滤或输出。相反，它维护了一个过滤器`logger`的列表和一个处理程序`handler`的列表。
2. 对于每条日志消息，记录器都会调用它的每个过滤器。如果有任何过滤器拒绝该消息，该消息就会被丢弃。
3. 对于每条被所有过滤器接受的日志消息，日志记录器会在其输出处理程序`hander`上循环，并要求每一个处理程序 `emit()` 该消息。

或者，至少，这就是这个想法的核心。标准库的日志记录实际上更加复杂。例如，每个处理程序可以携带它自己的过滤器列表，除了那些由它的记录器列出的过滤器。每个处理程序还指定了一个最小的消息 "level"，比如 `INFO` 或 `WARNING` ，相当令人困惑的是，这个级别既不是由处理程序本身也不是由处理程序的任何过滤器来执行的，而是由埋在日志记录器深处的 `if` 语句来执行的，它在处理程序中循环。因此，整个设计是有点混乱的。

但我们可以使用标准库记录器的基本见解——一个记录器的消息可能同时值得多个过滤器和多个输出——来完全解耦过滤器类和处理程序类。

```python
# There is now only one logger.
class Logger:
    def __init__(self, filters, handlers):
        self.filters = filters
        self.handlers = handlers

    def log(self, message):
        if all(f.match(message) for f in self.filters):
            for h in self.handlers:
                h.emit(message)

# Filters now know only about strings!
class TextFilter:
    def __init__(self, pattern):
        self.pattern = pattern

    def match(self, text):
        return self.pattern in text

# Handlers look like “loggers” did in the previous solution.
class FileHandler:
    def __init__(self, file):
        self.file = file

    def emit(self, message):
        self.file.write(message + '\n')
        self.file.flush()

class SocketHandler:
    def __init__(self, sock):
        self.sock = sock

    def emit(self, message):
        self.sock.sendall((message + '\n').encode('ascii'))

class SyslogHandler:
    def __init__(self, priority):
        self.priority = priority

    def emit(self, message):
        syslog.syslog(self.priority, message)
```

请注意，只有在我们设计的最后一个支点上，过滤器才真正以其应有的简单性闪亮登场。这是第一次，它们只接受一个字符串，只返回一个判决。之前所有的设计要么是将过滤功能隐藏在日志类中，要么是让过滤器承担了超出简单判决的额外职责。

事实上，"日志 "这个词已经从过滤器类的名称中完全删除了，而且有一个非常重要的原因：它不再有任何特定于日志的东西了。`TextFilter` 现在完全可以在任何涉及到字符串的情况下重用。最后与日志的具体概念解耦，它将更容易测试和维护。

同样的，就像所有解决一个问题的 _Composition Over Inheritance_ 方案一样，类在运行时被组成，而不需要任何继承性。

```python
f = TextFilter('Error')
h = FileHandler(sys.stdout)
logger = Logger([f], [h])

logger.log('Ignored: this will not be logged')
logger.log('Error: this is important')
```
```
Error: this is important
```

这里有一个关键的教训：像 _Composition Over Inheritance_ 这样的设计原则，最终要比  _Adapter_ 或 _Decorator_ 这样的个别模式更重要。始终遵循原则。但不要总是觉得被限制在官方列表中选择一个模式。我们现在得出的设计比以前的任何一个设计都更灵活，也更容易维护，尽管它们是基于官方的四人帮模式，但这个最终的设计却不是。有时候，是的，你会发现一个现有的设计模式完全适合你的问题--但如果不是，如果你超越它们，你的设计可能会更强大。

## Dodge:"if" 语句

我怀疑上面的代码让许多读者感到惊奇。对于一个典型的Python程序员来说，如此大量地使用类可能看起来完全是矫揉造作的——这是一种尴尬的练习，试图使20世纪80年代的旧思想看起来与现代Python有关。

当一个新的设计要求出现时，典型的 Python 程序员真的会去写一个新的类吗？没有！**"简单总比复杂好"**。为什么要增加一个类，如果一个 `if` 语句可以代替的话？一个单一的记录器类可以逐渐增加条件，直到它处理所有与我们之前的例子相同的情况:
```python
# Each new feature as an “if” statement.
class Logger:
    def __init__(self, pattern=None, file=None, sock=None, priority=None):
        self.pattern = pattern
        self.file = file
        self.sock = sock
        self.priority = priority

    def log(self, message):
        if self.pattern is not None:
            if self.pattern not in message:
                return
        if self.file is not None:
            self.file.write(message + '\n')
            self.file.flush()
        if self.sock is not None:
            self.sock.sendall((message + '\n').encode('ascii'))
        if self.priority is not None:
            syslog.syslog(self.priority, message)

# Works just fine.
logger = Logger(pattern='Error', file=sys.stdout)
logger.log('Warning: not that important')
logger.log('Error: this is important')
```

你可能认识到这个例子是你在实际应用中遇到的比较典型的 Python 设计实践。

`if` 语句的方法并非完全没有好处。这个类的所有可能的行为都可以在从上到下阅读代码的过程中掌握。参数列表可能看起来很冗长，但是由于 Python 的可选关键字参数，对这个类的大多数调用不需要提供所有的四个参数。

(这个类确实只能处理一个文件和一个套接字，但这是为了可读性而附带进行的简化。我们可以很容易地将 `file` 和 `socket` 的参数转为命名 `files` 和 `sockets` 的列表）。

鉴于每个Python程序员都能很快学会 `if`，但理解类却需要更长的时间，对于代码来说，依靠最简单的机制来实现一个功能，似乎是一个明显的胜利。但是让我们来平衡一下这种诱惑，明确说明躲避 _Composition Over Inheritance_ ("组合而非继承 ")会失去什么。

1. **Locality**，本地性。重组代码以使用 `if` 语句并没有为可读性带来无尽的好处。如果你的任务是改进或调试一个特定的功能——比如说，支持向套接字写入——你会发现你无法在一个地方阅读它的代码。这个单一功能背后的代码散落在初始化器的参数列表、初始化器的代码和 `log()` 方法本身之间。
2. **Deletability**，可删除性。好的设计的一个未被重视的特性是它使删除功能变得容易。也许只有大型和成熟的Python应用程序的老手才会强烈地体会到删除代码对项目健康的重要性。在我们基于类的解决方案中，我们可以通过删除 `SocketHandler` 类和它的单元测试，在应用程序不再需要它的时候，轻而易举地删除像记录到套接字的功能。相比之下，从森林中的 `if` 语句中删除socket功能不仅需要谨慎，以避免破坏相邻的代码，而且还提出了一个尴尬的问题：如何处理初始化器中的 `socket` 参数。它可以被删除吗？如果我们需要保持位置参数列表的一致性，那就不行了--我们需要保留这个参数，但如果它被使用就会引发一个异常。
3. **Dead code analysis**，死代码分析。与上一点相关的是，当我们使用 _Composition Over Inheritance_时，死代码分析器可以简单地检测到代码库中最后一次使用 `SocketHandler` 的时间消失。但死代码分析往往无能为力，无法做出 "你现在可以删除所有与 `socket` 输出有关的属性和 `if` 语句，因为没有任何幸存的初始化器调用为 `socket` 传递`None` 以外的东西 "的判断。
4. **Testing**，测试。我们的测试所提供的关于代码健康的最强烈的信号之一是，在到达被测试行之前，有多少行不相关的代码需要运行。如果测试可以简单地启动一个 `SocketHandler` 实例，传递给它一个活的套接字，并要求它 `emit()` 一个消息，那么测试一个像套接字日志这样的功能就很容易。除了与该功能相关的代码外，没有其他代码运行。但是在我们的 `if` 语句森林中测试套接字日志，至少要运行三倍的代码行数。仅仅为了测试其中的一个功能，就必须设置一个具有正确组合的记录器，这是一个重要的警告信号，在这个小例子中可能看起来微不足道，但随着系统的扩大，这一点变得至关重要。
5. **Efficiency**， 效率。我特意把这一点放在最后，因为可读性和可维护性通常是更重要的关注点。但是 `if` 语句之林的设计问题也体现在这种方法的低效率上。即使你想要一个简单的未经过滤的日志到一个文件中，每一条信息都将被迫针对你可能启用的每一个功能运行一个`if` 语句。相比之下，组合技术只对你所组合的功能运行代码。

由于所有这些原因，我建议，从软件设计的角度来看， `if` 语句森林的表面简单性在很大程度上是一种错觉。将记录器自上而下地读成一段代码的能力是以其他几种概念上的花费为代价的，这些花费会随着代码库的大小而急剧增长。

## Dodge: 多重继承

一些 Python 项目没有实践 _Composition Over Inheritance_，因为他们想通过 Python 语言的一个有争议的特性来回避这个原则：多重继承。

让我们回到我们开始时的例子代码，`FilteredLogger` 和 `SocketLogger` 是一个基础 `Logger` 类的两个不同子类。在一个只支持单继承的语言中，`FilteredSocketLogger` 必须选择从 `SocketLogger` 或 `FilteredLogger` 继承，然后必须重复另一个类的代码。

但是 Python 支持多重继承，所以新的 `FilteredSocketLogger`可以将 `SocketLogger` 和 `FilteredLogger` 都列为基类，并从两者继承。

```python
# Our original example’s base class and subclasses.
class Logger(object):
    def __init__(self, file):
        self.file = file

    def log(self, message):
        self.file.write(message + '\n')
        self.file.flush()

class SocketLogger(Logger):
    def __init__(self, sock):
        self.sock = sock

    def log(self, message):
        self.sock.sendall((message + '\n').encode('ascii'))

class FilteredLogger(Logger):
    def __init__(self, pattern, file):
        self.pattern = pattern
        super().__init__(file)

    def log(self, message):
        if self.pattern in message:
            super().log(message)

# A class derived through multiple inheritance.
class FilteredSocketLogger(FilteredLogger, SocketLogger):
    def __init__(self, pattern, sock):
        FilteredLogger.__init__(self, pattern, None)
        SocketLogger.__init__(self, sock)

# Works just fine.

logger = FilteredSocketLogger('Error', sock1)
logger.log('Warning: not that important')
logger.log('Error: this is important')

print('The socket received: %r' % sock2.recv(512))
```
```
The socket received: b'Error: this is important\n'
```

这与我们的 _Decorator Pattern_ 解决方案有几处惊人的相似之处。在这两种情况下。

1. 每种输出都有一个记录器类（而不是我们的 `Adapter` 在直接写文件和通过`Adapter`写非文件之间的不对称）。
2. `message` 保留了由调用者提供的精确值（而不是我们的`Adapter`习惯性地通过添加换行来替换它的文件特定值）。
3. 过滤器和记录器是对称的，因为它们都实现了相同的方法 `log()`。(除了 _Decorator_ 之外，我们的其他解决方案是过滤器类提供一种方法，而输出类提供另一种方法)。
4. 过滤器从不尝试自己产生输出，但如果一条消息在过滤过程中幸存下来，就会把输出的任务推迟到其他代码。

这些与我们先前的 _Decorator_ 解决方案的密切相似性意味着我们可以将其与这段新代码进行比较，从而在 _Composition Over Inheritance_ 和多重继承之间进行异常鲜明的比较。让我们用一个问题进一步突出重点。

__如果我们对记录器和过滤器都进行了彻底的单元测试，那么我们有多大把握让它们一起工作？__

1. _Decorator_ 例子的成功只取决于每个类的公共行为：`LogFilter` 提供了一个`log()` 方法，该方法反过来调用它所包装的对象上的`log()` （测试可以用一个微小的假记录器简单地验证），并且每个记录器提供了一个有效的`log()` 方法。只要我们的单元测试验证了这两个公共行为，我们就不能在不通过单元测试的情况下破坏组合。

    相比之下，多重继承依赖于那些无法通过简单实例化相关类来验证的行为。`FilteredLogger` 的公共行为是，它提供了一个`log()`方法，既能过滤又能写入文件。但是多重继承并不仅仅取决于公共行为，而是取决于该行为在内部的实现方式。如果该方法使用`super()` 定义其基类，那么多重继承将起作用，但如果该方法自己`write()` 到文件，则不会起作用，尽管这两种实现都能满足单元测试的要求。

    因此，测试套件必须超越单元测试，对类进行实际的多重继承--或者用猴子补丁来验证`log()`调用`super().log()`——以保证多重继承在未来的开发者对代码的工作中保持有效。

3. 多重继承引入了一个新的 `__init__()` 方法，因为基类的` __init__() `方法都不接受足够的参数用于组合过滤器和日志器。这段新的代码需要被测试，所以每个新的子类至少需要一个测试。

    你可能会被诱惑去构思一个方案，以避免每个子类都有一个新的 `__init__()` ，比如接受 `*args` 然后将它们传递给 `super().__init__()`。(如果你真的采用这种方法，请回顾一下经典的文章 "[Python’s Super Considered Harmful](https://fuhm.net/super-harmful/)"，它论证了只有`**kwargs` 实际上是安全的。) 这种方案的问题在于它损害了可读性--你不能再仅仅通过阅读参数列表来弄清`__init__()` 方法需要哪些参数。而且类型检查工具将不再能够保证正确性。

    但是无论你给每个派生类提供它自己的` __init__() `方法，还是把它们设计成链状，你对原始的 `FilteredLogger` 和 `SocketLogger` 的单元测试本身都不能保证这些类在组合时初始化正确。

    相比之下，_Decorator_ 的设计让它的初始化器愉快地、严格地正交。过滤器接受它的`pattern`，记录器接受它的`sock`，两者之间不可能有冲突。

4. 最后，有可能两个类本身工作得很好，但有相同名称的类或实例属性，当这些类通过多重继承组合在一起时，就会发生冲突。

    是的，我们这里的小例子使碰撞的几率看起来很小，不用担心——但请记住，这些例子只是代表了你在实际应用中可能写的复杂得多的类。

    无论程序员是通过在每个类的实例上运行 `dir()` 并检查它们的共同属性来编写防止碰撞的测试，还是为每个可能的子类编写集成测试，两个独立类的原始单元测试将再次无法保证它们能通过多重继承干净地结合起来。

由于上述任何一个原因，两个基类的单元测试可以保持绿色，即使它们通过多重继承组合的能力被破坏。这意味着四人帮的 "支持每一种组合的子类爆炸 "也会影响你的测试。只有通过测试应用程序中 `m×n` 个基类的每个组合，你才能使应用程序在运行时安全地使用这些类。

除了破坏单元测试的保证外，多重继承还涉及至少三个进一步的责任。

4. 在  _Decorator_ 的情况下，自省很简单。只需`print(my_filter.logger)`或在调试器中查看该属性，就可以看到附加了什么样的输出记录器。然而，在多重继承的情况下，你只能通过检查类本身的元数据来了解哪个`filter`和`logger`被结合在一起——通过读取它的 `__mro__ `或者对对象进行一系列的 `isinstance()` 测试。
5. 在 _Decorator_ 的情况下，将一个 过滤器 `filter`和记录器 `logger` 的实时组合，并在运行时通过对 `.logger` 属性的赋值换入一个不同的记录器——比如说，因为用户刚刚在应用程序的界面中切换了一个偏好，这是非常简单的。但在多继承的情况下，要做同样的事情，就需要采取更令人反感的手法，即覆盖对象的类。虽然在Python这样的动态语言中，在运行时改变一个对象的类并非不可能，但这通常被认为是软件设计出了问题的表现。
6. 最后，多重继承没有提供内置的机制来帮助程序员正确排列基类。如果 `FilteredSocketLogger` 的基类被调换，它就不能成功地写入一个套接字，而且正如Stack Overflow上的许多问题所证明的那样，Python程序员在把第三方基类放在正确的顺序上方面一直有困难。相比之下，_Decorator_ 模式使类的组成方式显而易见：过滤器的 `__init__() `想要一个`logger` 对象，但记录器的 `__init__()` 并不要求一个`filter`

那么，多重继承就会产生一些责任，而没有增加任何优势。至少在这个例子中，用继承来解决设计问题，严格来说比基于组合的设计要差。

## Dodge: 混合

 上一节中的 `FilteredSocketLogger` 需要它自己的自定义 `_init__()` 方法，因为它需要接受其两个基类的参数。但事实证明，这种责任是可以避免的。当然，在子类不需要任何额外数据的情况下，这个问题就不会出现。但即使是需要额外数据的类，也可以通过其他方式来传递数据。

如果我们在类本身中为`pattern`提供一个默认值，然后邀请调用者在初始化的范围之外直接定制该属性，我们可以使`FilteredLogger`对多重继承更加友好。

```python
# Don’t accept a “pattern” during initialization.
class FilteredLogger(Logger):
    pattern = ''

    def log(self, message):
        if self.pattern in message:
            super().log(message)

# Multiple inheritance is now simpler.
class FilteredSocketLogger(FilteredLogger, SocketLogger):
    pass  # This subclass needs no extra code!

# The caller can just set “pattern” directly.
logger = FilteredSocketLogger(sock1)
logger.pattern = 'Error'

# Works just fine.
logger.log('Warning: not that important')
logger.log('Error: this is important')

print('The socket received: %r' % sock2.recv(512))
```
```
The socket received: b'Error: this is important\n'
```
在将 `FilteredLogger` 转向与其基类正交的初始化手法之后，为什么不将正交的想法推向其逻辑结论呢？我们可以将 `FilteredLogger` 转换为一个 "mixin(混合器)"，它完全生活在类的层次结构之外，多重继承将使它与之结合。

```python
# Simplify the filter by making it a mixin.
class FilterMixin:  # No base class!
    pattern = ''

    def log(self, message):
        if self.pattern in message:
            super().log(message)

# Multiple inheritance looks the same as above.
class FilteredLogger(FilterMixin, FileLogger):
    pass  # Again, the subclass needs no extra code.

# Works just fine.
logger = FilteredLogger(sys.stdout)
logger.pattern = 'Error'
logger.log('Warning: not that important')
logger.log('Error: this is important')
```
```
Error: this is important
```

_mixin_ 在概念上比我们在上一节看到的过滤子类更简单：它没有可能使方法解析顺序复杂化的基类，所以`super()`将总是调用类声明中列出的下一个基类。

与同等的子类相比，_mixin_ 也有一个更简单的测试故事。`FilteredLogger` 需要测试它独立运行并与其他类结合，而`FilterMixin` 只需要测试它与记录器的结合。因为 _mixin_ 本身是不完整的，甚至不能写一个独立运行的测试。

但所有其他的多重继承的责任仍然适用。因此，虽然 _mixin_ 模式确实改善了多重继承的可读性和概念上的简单性，但它并不是解决其问题的完整方案。

## Dodge: 动态建类

正如我们在前两节所看到的，无论是传统的多重继承还是混搭，都没有解决四人帮的 "支持每一种组合的子类爆炸 "的问题--它们只是在需要组合两个类的时候避免了代码的重复。

在一般情况下，多重继承仍然需要 "大量的类"，有 _m×n_ 个类声明，每个声明看起来都是这样。

```python
class FilteredSocketLogger(FilteredLogger, SocketLogger):
    ...
```

但事实证明，Python提供了一个变通方法。

想象一下，我们的应用程序读取一个配置文件来学习它应该使用的日志过滤器和日志目的地，这个文件的内容在运行时才会知道。与其提前建立所有 _m×n_ 个可能的类，然后选择正确的一个，我们可以等待，并利用 Python 不仅支持类声明，还支持一个内置的 `type()` 函数，在运行时动态地创建新的类。

```python
# Imagine 2 filtered loggers and 3 output loggers.

filters = {
    'pattern': PatternFilteredLog,
    'severity': SeverityFilteredLog,
}
outputs = {
    'file': FileLog,
    'socket': SocketLog,
    'syslog': SyslogLog,
}

# Select the two classes we want to combine.

with open('config') as f:
    filter_name, output_name = f.read().split()

filter_cls = filters[filter_name]
output_cls = outputs[output_name]

# Build a new derived class (!)

name = filter_name.title() + output_name.title() + 'Log'
cls = type(name, (filter_cls, output_cls), {})

# Call it as usual to produce an instance.

logger = cls(...)
```

传递给 `type()`的类的元组与类声明中的一系列基类具有相同的意义。上面的 `type()`调用通过对过滤型记录器和输出型记录器的多重继承创建了一个新的类。

在你问之前：是的，以纯文本形式构建类声明，然后将其传递给`eval()`，也是可以的。

但即时构建类会带来严重的后果。

- 可读性受到影响。一个人在阅读上面的代码片段时，必须做额外的工作来确定 `cls` 的实例是什么样的对象。另外，许多 Python 程序员并不熟悉 `type()` ，他们需要停下来仔细研究其文档。如果他们对类可以被动态定义这一新颖的概念有困难，他们可能还会感到困惑。
- 如果一个像`PatternFilteredFileLog` 这样的构造类在异常或错误信息中被命名，开发者可能会不高兴地发现，当他们在代码中搜索这个类的名字时，什么都没有出现。当你甚至无法找到一个类的时候，调试就变得更加困难。相当多的时间可能被用来搜索代码库中的`type()`调用，并试图确定哪一个生成了这个类。有时，开发者不得不用坏的参数来调用每个方法，并使用所产生的回溯中的行号来追踪基类的位置。
- 在一般情况下，对于在运行时动态构建的类，类型自省会失败。当你在调试器中高亮显示`PatternFilteredFileLog`的实例时，编辑器中的 "跳转到类 "快捷键将无法带你到任何地方。像 [mypy](https://github.com/python/mypy) 和 [pyre-check](https://github.com/facebook/pyre-check) 这样的类型检查引擎将不可能为你生成的类提供强大的保护，因为它们能够为普通的 Python 类提供保护。
- 美丽的 Jupyter 笔记本功能 `%autoreload` 拥有一种近乎超自然的能力，可以在实时 Python 解释器中检测并重新加载修改过的源代码。但是它被挫败了，例如，[matplotlib 在运行时](https://github.com/matplotlib/matplotlib/blob/54b426397c0e7567edaee4f7f77036c2b8569573/lib/matplotlib/axes/_subplots.py#L180) 通过`subplot_class_factory()` 中的`type()`调用建立的多个继承类。

一旦权衡了它的责任，试图使用运行时类的生成作为最后的手段来挽救已经有问题的多重继承机制的做法，是对当你需要一个对象的行为在几个独立的轴上变化时，躲避组成大于继承的整个项目的一种归纳和荒谬。
