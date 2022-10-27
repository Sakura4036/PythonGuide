# 语言规范
参考：[Google Python语言规范 ](https://zh-google-styleguide.readthedocs.io/en/latest/google-python-styleguide/python_language_rules/#)

## 导入
导入总应该放在文件顶部，位于模块注释和文档字符串之后，模块全局变量和常量之前。导入应该按照从最通用到最不通用的顺序分组：

 1. Future 导入语句：
```python
from __future__ import print_function
```
 2. 标准库导入：
```python
import sys
```
 3. [第三方](https://pypi.org/)模块或包导入：
```python
import tensorflow as tf
```
 4. 代码库子包导入：
```python
from otherproject.ai import mind
```
 5. **已弃用：** 与此文件属于同一顶级子包的应用程序特定导入。例如：
```python
from myproject.backend.hgwells import time_machine
``` 
您可能会发现之前的 Google Python 风格是这么做的，但现在已经不推荐了。**新的代码不要这么做**。只需将特定于应用程序的子包导入与其他子包导入一样对待即可。

**强烈建议**：

 - 仅对包和模块使用导入,而不单独导入函数或者类。`typing`模块例外
 - 使用模块的**全路径名**来导入每个模块
 - 导入时不要使用相对名称 (`from .x import y`)。 即使模块在同一个包中, 也要使用完整包名。
 - 仅当缩写  `z`  是通用缩写时才可使用  `import  y  as  z`.(比如  `np`  代表  `numpy`)

## 异常
正常操作代码的控制流不会和错误处理代码混在一起。当某种条件发生时，它也允许控制流跳过多个框架。例如, 一步跳出N个嵌套的函数，而不必继续执行错误的代码。

使用以下```try ... execpt ...```语句来捕获可能发生的异常。
```python
try:
	xxx
except Exception as e:
	xxx
else: 
	# 没有捕获到异常才执行
	xxx

# 或者finally
try:
	xxx
except Exception as e:
	xxx
finally: 
	# 无论有没有捕获到异常都执行
	xxx
```
**使用异常必须遵守特定条件：**

 - 优先合理的使用内置异常类， 例如`ValueError`，`TypeError`，`OSError`等等。
 - 不要使用 `assert` 语句来验证公共API的参数值. `assert` 是用来保证内部正确性的,而不是用来强制纠正参数使用，这种情况应该使用`raise`语句来抛出异常。
 - 自定义异常应继承内建的`Exception`类，且异常类的命名后缀为**Error**， 如`ReadError`等。
 - 永远不要使用 `except:` 语句来捕获所有异常, 也不要捕获 `Exception` 或者 `StandardError`。 否则python会捕获**所有异常**（包括**语法错误**），因此使用``except:`` 或者``except Exception:``，很容易隐藏真正的**Bug**，并且不容易定位查找。
 -  尽量减少``try/except``块中的代码量。``try``块的体积越大, 期望之外的异常就越容易被触发. 这种情况下, ``try/except``块将隐藏真正的错误
 - 使用finally子句来执行那些无论try块中有没有异常都应该被执行的代码. 这对于清理资源常常很有用, 例如关闭文件
 
 以上知识对异常的简单说明，详细说明与高阶使用技巧见：[Python 项目工程化开发指南-异常](https://pyloong.github.io/pythonic-project-guidelines/guidelines/advanced/exception/)。
 
## 全局变量
 
虽然全局变量有时很方便很好用，但是导入时可能改变模块行为，因为导入模块时会对模块级变量赋值。例如C模块从A模块中导入B变量，在C模块中修改B变量会影响A中的B变量，作用等同于函数中修改可变参数。（**注**：这里的变量指的是python中的可变类型。）
 
 - 鼓励使用模块级的常量，且常量命名必须全部大写，使用`_`进行分隔。
 - 若必须要使用全局变量，应在模块内声明全局变量,并在名称前 `_` 使之成为模块内部变量.外部访问必须通过模块级的公共函数。

##  嵌套/局部/内部类或函数（进阶技巧）
**前提：**

 - 类可以定义在方法, 函数或者类中
 - 函数可以定义在方法或函数中
 - 不可变类型的变量对嵌套函数是只读的。即内嵌函数可以读外部函数中定义的变量,但是无法改写,除非使用 `nonlocal`对该变量进行声明
 - 内嵌函数和类会使外部函数的**可读性变差**
 
 使用内部类或者内嵌函数可以忽视一些警告。但是应该避免使用内嵌函数或类，除非是想覆盖某些值。
 若想对模块的用户隐藏某个函数，不要采用嵌套它来隐藏，应该在需要被隐藏的方法的模块级名称加 `_` 前缀,这样它依然是可以被测试的。
 
```python
# test 1
def func():
	i = 1
	l = [1,2]
	def anonymous_func():
		# i is readable only
		# 因此对i的操作不影响外部的i
		print(i) # i=1
		l.append(1) # l是list，属于可变类型
		print(l) # l= [1,2,1]
	print(i) # i=1
	print(l) # l= [1,2,1]
# test 2
def func():
	i = 1
	def anonymous_func():
		nonlocal i
		i = 2 
		print(i) # i=2
	print(i) # i=2
```

## 推导式与生成式

列表，字典和集合的推导与生成式提供了一种简洁高效的方式来创建容器和迭代器, 而不必借助`map()`，`filter()`，或者`lambda`。元组是没有推导式的, `()` 内加类似推导式的句式返回的是个生成器。

 - 可以在简单情况下使用
 - 每个部分应该单独置于一行：映射表达式，for语句，过滤器表达式
 - 禁止多重for语句或过滤器表达式，复杂情况下还是使用循环

 **Good**
```python
result = [mapping_expr for value in iterable if filter_expr]

values = [func(value) for value in values if filter(value)]

squares_generator = (x**2 for x in range(10))

adult_names = {user["name"] for user in users if user['age'] >= 18}
```
**Bad**
```python
result = [(x, y) for x in range(10) for y in range(5) if x * y > 10]

return ((x, y, z)
        for x in xrange(5)
        for y in xrange(5)
        if x != y
        for z in xrange(5)
        if y != z)
```

## 迭代器与操作符
容器类型，像字典和列表，定义了默认的迭代器和关系测试操作符(`in`和`not in`)。它们直接表达了操作，没有额外的方法调用。 使用默认操作符的函数是通用的。 它可以用于支持该操作的任何类型。

**Good**
```python
for key in adict: ...
if obj in alist: ...
for line in afile: ...
# 迭代器，一次一行地读取文件
for k, v in dict.iteritems(): ...
```
**Bad**
```python
for key in adict.keys(): ...
if not adict.has_key(key): ...
for line in afile.readlines(): ... 
# readlines()会读取全部文件加载到内存
```

## 生成器
生成器可以看作是一种特殊的迭代器。
所谓生成器（函数），就是每当它执行一次生成`yield`语句，它就返回一个迭代器，这个迭代器生成一个值。生成值后，生成器函数的运行状态将被挂起，直到下一次生成。

```python
# example
def get_batch(dataset):
	for data in dataset:
		batch_data = process(data)
		yield batch_data

for batch in get_batch(dataset):
	output = model(batch)
	...
```
## Lambda
lambda在一个表达式中定义匿名函数. 常用于为 `map()` 和 `filter()` 之类的高阶函数定义回调函数或者操作符。

 - 适用于单行函数，一般不超过60-80个字符
 - 没有函数名意味着堆栈跟踪更难理解， 难以阅读和调试
 -  由于lambda函数通常只包含一个表达式，因此其表达能力有限
 - 对于常见的操作符，例如乘法操作符，使用 `operator` 模块中的函数以代替lambda函数. 例如, 推荐使用 `operator.mul` , 而不是 `lambda x, y: x * y`
 
```python
# example
sorted_values = sorted(values, key=lambda x: x[0])
```

## 条件表达式
条件表达式(又名三元运算符)是对于if语句的一种更为简短的句法规则. 例如: `x = 1 if cond else 2` 。
写法上推荐真实表达式，`if`表达式，`else`表达式每个独占一行。在其他情况下，推荐使用完整的if语句。

```python
device = torch.device('cuda') if torch.cuda.is_available() else torch.device('cpu')
# or
device = torch.device('cuda'if torch.cuda.is_available() else 'cpu')
```
## 默认参数值
你可以在函数参数列表的最后指定变量的值, 例如，`def foo(a, b = 0):` 。如果调用foo时只带一个参数，则b被设为0。如果带两个参数，则b的值等于第二个参数。

**注**：默认参数只在模块加载时求值一次。如果参数是列表或字典之类的可变类型，这可能会导致问题。如果函数修改了对象（例如向列表追加项），默认值就被修改了。

**Good：**
```python
def foo(a, b:List[int]=None):
    if b is None:
        b = []
    
# Empty tuple OK since tuples are immutable
def foo(a, b: Sequence = ()):  
```
**Bad：**
```python
def foo(a, b=[]):
def foo(a, b={}):
def foo(a, b=time.time()):
```

## True与False
Python在布尔上下文中会将某些值求值为`False`。按简单的直觉来讲，就是所有的”空”值都被认为是`False`。
`0`，`[]`，`{}`，`""`都被认为是`False`。
**注意：False is not None**，使用`if not x`与`if x is not None`是不同的判断条件，注意使用情况。

 - 尽可能使用隐式的false, 例如: 使用 `if foo:` 而不是 `if foo != []:`
 - 永远不要用==将一个布尔量与`False`相比较。使用 `if  not x:` 代替。 如果你需要区分`False`和`None`, 你应该用像 `if not x and x is not None:` 这样的语句。
 - 对于序列(字符串，列表，元组)，`if  not seq:`  或者  `if seq:`  比  `if len(seq):`  或  `if not len(seq):`  要更好。
 - 处理整数时，使用隐式`False`可能会得不偿失(即不小心将`None`当做0来处理)。你可以将一个已知是整型(且不是`len()`的返回结果)的值与0比较.、
 - 注意`’0’`(字符串)会被当做`True`

**Good：**
```python
if not users:
    print('no users')

if foo == 0:
    self.handle_zero()

if i % 10 == 0:
    self.handle_multiple_of_ten()

def f(x=None):
    if x is None:
        x = []
```
**Bad：**
```python
if len(users) == 0:
    print 'no users'

if foo is not None and not foo: #条件判断顺序反了
    self.handle_zero()

if not i % 10:
    self.handle_multiple_of_ten()

def f(x=None):
    x = x or []
```
 
 #### 过时的语言特性

 - 尽可能使用字符串方法取代字符串模块
 - 使用函数调用语法取代`apply()`
 - 使用列表推导式，`for`循环取代`filter()`，`map()`以及`reduce()`。

## 方法装饰器

[用于函数及方法的装饰器](https://docs.python.org/release/2.4.3/whatsnew/node6.html) (也就是@标记)。 最常见的装饰器是`@classmethod` 和`@staticmethod`，用于将常规函数转换成类方法或静态方法。
不过，装饰器语法也允许用户自定义装饰器。特别地，对于某个函数 `my_decorator` ，下面的两段代码是等效的：
```python
class C(object):
   @my_decorator
   def method(self):
       # method body ...

# equal to 
class C(object):
    def method(self):
        # method body ...
    method = my_decorator(method)
```
装饰器可以在函数被执行之前先做一下预处理，也就是在函数已有的功能上添加额外的功能，但保持已有函数不变（enforce invariant）。对于经常调用的函数，可以定义为装饰器，减少一些重复代码，例如：`login_required`，`check_value`等等。

 - 如果好处很显然, 就明智而谨慎的使用装饰器
 - 使用装饰器时，文档应该清晰的说明该函数是一个装饰器
 - 除非是为了将方法和现有的API集成，否则不要使用 `staticmethod` 。多数情况下，将方法封装成模块级的函数可以达到同样的效果。
 - 谨慎使用 `classmethod`，通常只在定义备选构造函数，或者写用于修改诸如进程级缓存等必要的全局状态的特定类方法才用。

## 代码类型注释
你可以根据 [PEP-484](https://www.python.org/dev/peps/pep-0484/) 来对 python3 代码进行注释,并使用诸如 [pytype](https://github.com/google/pytype) 之类的类型检查工具来检查代码。 类型注释既可以写在源码，也可以写在 [pyi](https://www.python.org/dev/peps/pep-0484/#stub-files) 中。推荐尽量写在源码里，对于第三方扩展包，可以写在pyi文件里。详情见（[风格注释 - 类型注释](../style_rules/#_12)）

用于函数参数和返回值的类型注释：
```python
def func(a: int) -> List[int]:
def func(s:str, ls:Optional[str,List[str]]=None)-> str
```
