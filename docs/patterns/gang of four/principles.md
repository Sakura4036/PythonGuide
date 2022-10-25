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

## 方案#1: 适配器（Adapter）模式


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

## 装饰器（Decorator）模式

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


