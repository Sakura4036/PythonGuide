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

## 方案4 更优

Python的日志模块想要更多的灵活性：不仅要支持多个过滤器，而且要支持单个日志信息流的多个输出。基于其他语言的日志模块的设计--主要的灵感请参见`PEP 282`的 "影响 "部分——Python的日志模块实现了自己的 `Composition Over Inheritance` 模式。

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
