# 风格规范

参考：[Python 风格规范](https://zh-google-styleguide.readthedocs.io/en/latest/google-python-styleguide/python_style_rules/)

## 行长度
一行代码的长度默认不超过80个字符（含空格）。由于现在显示器更大的尺寸和更高的像素，这个规范有点过时，因此一行不超过120个字符也是可以接受的。

以下场景是例外：

 1. 长的`import` 语句，例如从某一模块中调用多个函数等
 2. 注释中的URL， 文件路径，或一些特定的长Token
 3. 不便于换行，不包含空格的模块级字符串常量，比如URL或者路径

Python会将**圆括号、中括号和花括号**中的行**隐式地连接**起来：
```python
def foo_bar(self, width, height, color='black', design=None, x='foo',
        emphasis=None, highlight=0)
 
 x = ('This will build a very long long '
     'long long long long long long string')
 
# http://www.example.com/us/developer/documentation/api/content/v2.0/csv_file_name_extension_full_specification.html
```

## 括号
宁缺毋滥的使用括号。
除非是用于实现行连接或者元组， 否则不要在返回语句或条件语句中使用括号。

**Good**
```python
if x and y:

return value


return spam, beans  # 返回2个对象
return (spam, beans)  # 返回1个元组 

x = ('This will build a very long long '
     'long long long long long long string')
```
**Bad**
```python
if (x):
return (foo)
```


## 缩进

 - 用4个空格来缩进代码
 - 除非在IDE中，否则不要使用Tab
 - 也不要Tab和空格混用
 - 多行时，注意换行垂直对齐
 
**Good**
```python
# Aligned with opening delimiter
foo = long_function_name(var_one, var_two,
                         var_three, var_four)

# Aligned with opening delimiter in a dictionary
foo = {
    long_dictionary_key: value1 +
                         value2,
    ...
}

# 4-space hanging indent in a dictionary
foo = {
    long_dictionary_key:
        long_dictionary_value,
    ...
}
```

**Bad**
```python
# Stuff on first line forbidden
foo = long_function_name(var_one, var_two,
    var_three, var_four)

# No hanging indent in a dictionary
foo = {
    long_dictionary_key:
    long_dictionary_value,
    ...
}
```

## 序列元素尾部逗号
仅当 `]`, `)`, `}` 和末位元素不在同一行时，推荐使用序列元素尾部逗号. 当末位元素尾部有逗号时，元素后的逗号可以指示 [YAPF](https://pypi.org/project/yapf/) 将序列格式化为每行一项。

**Good**
```python
golomb4 = [  
    0,  
    4,  
    6,  
]
```

**Bad**
```python
golomb4 = [  
    0,  
    4,  
    6 
]
```

## 空行

 - 顶级定义之间空两行
 - 类方法定义之间空一行
 - 函数或方法中，某些地方要是你觉得合适，就空一行


## 注释

### 文档字符串（`__doc__`）
Python有一种独一无二的的注释方式：使用文档字符串。文档字符串是包，模块，类或函数里的第一个语句。这些字符串可以通过对象的 `__doc__` 成员被自动提取，并且被pydoc所用。(你可以在你的模块上运行pydoc试一把，看看它长什么样)。我们对文档字符串的惯例是使用三重双引号”””( [PEP-257](http://www.python.org/dev/peps/pep-0257/) )。 

一个文档字符串应该这样组织：首先是一行以句号，问号或惊叹号结尾的概述(或者该文档字符串单纯只有一行)。接着是一个空行。接着是文档字符串剩下的部分，它应该与文档字符串的第一行的第一个引号对齐。

### 模块
每个文件应该包含一个许可样板，根据项目使用的许可(例如，Apache 2.0，BSD，LGPL，GPL), 选择合适的样板。其开头应是对模块内容和用法的描述。多见于开源仓库。

### 函数与方法
下文所指的函数，包括函数，方法，以及生成器。

一个函数必须要有文档字符串，除非它满足以下条件:
 1. 外部不可见
 2.  非常短小
 3.  简单明了

文档字符串应该包含函数做什么，以及输入和输出的详细描述。通常，不应该描述“怎么做”或者实现细节，除非是一些复杂的算法。对于复杂的代码，在代码旁边加注释会比使用文档字符串更有意义。
覆盖基类的子类方法应有一个类似 `See base class` 的简单注释来指引读者到基类方法的文档注释。若重载的子类方法和基类方法有很大不同，那么注释中应该指明这些信息。

关于函数的几个方面应该在特定的小节中进行描述记录，这几个方面如下文所述。每节应该以一个标题行开始。标题行以冒号结尾。除标题行外，节的其他内容应被缩进2个空格。

**Args:**

列出每个参数的名字, 并在名字后使用一个冒号和一个空格, 分隔对该参数的描述.如果描述太长超过了单行最大长度（`80`或者自定义）字符数，使用2或者4个空格的悬挂缩进(与文件其他部分保持一致)。
描述应该包括所需的类型和含义。如果一个函数接受`*args`(可变长度参数列表)或者`**kwargs` (任意关键字参数), 应该详细列出`*args`和`**kwargs`。

**Returns: (或者 Yields: 用于生成器)**

描述返回值的类型和语义. 如果函数返回None, 这一部分可以省略.

**Raises:**

列出与接口有关的所有异常.

```python
def fetch_smalltable_rows(table_handle: smalltable.Table,
                          keys: Sequence[Union[bytes, str]],
                          require_all_keys: bool = False,
) -> Mapping[bytes, Tuple[str]]:
	"""Fetches rows from a Smalltable.

	Retrieves rows pertaining to the given keys from the Table instance
	represented by table_handle.  String keys will be UTF-8 encoded.

	Args:
		table_handle: An open smalltable.Table instance.
		keys: A sequence of strings representing the key of each table
		row to fetch.  String keys will be UTF-8 encoded.
		require_all_keys: Optional; If require_all_keys is True only
		rows with values set for all keys will be returned.

	Returns:
		A dict mapping keys to the corresponding table row data
		fetched. Each row is represented as a tuple of strings. For
		example:

		{b'Serak': ('Rigel VII', 'Preparer'),
		b'Zim': ('Irk', 'Invader'),
		b'Lrrr': ('Omicron Persei 8', 'Emperor')}

		Returned keys are always bytes.  If a key from the keys argument is
		missing from the dictionary, then that row was not found in the
		table (and require_all_keys must have been False).

	Raises:
		IOError: An error occurred accessing the smalltable.
 """
```

### 类注释
类应该在其定义下有一个用于描述该类的文档字符串。如果你的类有公共属性(Attributes)，那么文档中应该有一个属性(Attributes)段。并且应该遵守和函数参数相同的格式。
```python
class SampleClass(object):
    """Summary of class here.

	 Longer class information....
	 Longer class information....

	 Attributes:
	 likes_spam: A boolean indicating if we like SPAM or not.
	 eggs: An integer count of the eggs we have laid.
 """

    def __init__(self, likes_spam=False):
        """Inits SampleClass with blah."""
        self.likes_spam = likes_spam
        self.eggs = 0

    def public_method(self):
        """Performs operation blah."""
        ...
```

### 块注释与行注释
最需要写注释的是代码中那些技巧性的部分。如果你在下次 [代码审查](http://en.wikipedia.org/wiki/Code_review) 的时候必须解释一下，那么你应该现在就给它写注释。对于复杂的操作，应该在其操作开始前写上若干行注释， 多行注释可以使用```"""..."""```。对于不是一目了然的代码，应在其行尾添加注释。

```python
# We use a weighted dictionary search to find out where i is in
# the array.  We extrapolate position based on the largest num
# in the array and the array size and then do binary search to
# get the exact number.

if i & (i-1) == 0:  # True if i is 0 or a power of 2
```

为了提高可读性，注释应该至少离开代码2个空格。

另一方面，绝不要描述代码。假设阅读代码的人比你更懂Python，他只是不知道你的代码要做什么。
```python
# BAD COMMENT
# Now go through the b array and make sure whenever i occurs
# the next element is i+1
```

## 类型注释

### **通用规则**

1.  请先熟悉下 [PEP-484 ](https://www.python.org/dev/peps/pep-0484/)
2.  对于方法，仅在必要时才对  `self`  或  `cls`  注释
3.  若对类型没有任何显示，请使用  `Any`
4.  无需注释模块中的所有函数

	    1.  公共的API需要注释
	    2.  在代码的安全性，清晰性和灵活性上进行权衡是否注释
	    3.  对于容易出现类型相关的错误的代码进行注释
	    4.  难以理解的代码请进行注释
	    5.  若代码中的类型已经稳定，可以进行注释. 对于一份成熟的代码，多数情况下，即使注释了所有的函数，也不会丧失太多的灵活性。

### **换行**

尽量遵守既定的缩进规则。注释后，很多函数签名将会变成每行一个参数。
```python
def my_method(self,
              first_var: int,
              second_var: Foo,
              third_var: Optional[Bar]) -> int:
    ...
```
尽量在变量之间换行而不是在变量和类型注释之间.当然,若所有东西都在一行上,也可以接受
```python
def my_method(self, first_var: int) -> int:
	...
```
若是函数名,末位形参和返回值的类型注释太长,也可以进行换行,并在新行进行4格缩进.
```python
def my_method(
    self, first_var: int) -> Tuple[MyLongType1, MyLongType1]:
	...
```
若是末位形参和返回值类型注释不适合在同一行上,可以换行,缩进为4空格,并保持闭合的括号 `)` 和 `def` 对齐
```python
def my_method(
    self, other_arg: Optional[MyLongType]
) -> Dict[OtherLongType, MyLongType]:
	...
```
尽量不要在一个类型注释中进行换行.但是有时类型注释过长需要换行时,请尽量保持子类型中不被换行.
```python
def my_method(
    self,
    first_var: Tuple[List[MyLongType1],
                     List[MyLongType2]],
    second_var: List[Dict[
        MyLongType3, MyLongType4]]) -> None:
	...
```

### **参数默认值**
依据 [PEP-008](https://www.python.org/dev/peps/pep-0008/#other-recommendations) ,仅对同时具有类型注释和默认值的参数的 `=` 周围加空格。
```python
# good
def func(a: int = 0) -> int:
	...

# bad
def func(a:int=0) -> int:
	...
```

### **可选参数类型**
在python的类型系统中, `NoneType` 是 “一等对象”，为了输入方便，`None` 是 `NoneType` 的别名。一个变量若是 `None`，则该变量必须被声明。
因为python中函数参数可以是任何类型，若没有特殊声明则默认为`Any`类型。若函数参数有多个类型，我们可以使用 `typing`模块中的`Union`，但若类型仅仅只是对应另一个其他类型，建议使用 `Optional`。

尽量显式而非隐式的使用 `Optional`。在PEP-484的早期版本中允许使用 `a: Text = None` 来替代 `a: Optional[Text] = None`，当然,现在不推荐这么做了.

```python
# good
def func(a: Optional[Text], b: Optional[Text] = None) -> Text:
    ...
def multiple_nullable_union(a: Union[None, Text, int]) -> Text
    ...

# bad
def nullable_union(a: Union[None, Text]) -> Text:
    ...
def implicit_optional(a: Text = None) -> Text:
    ...
```
### **忽略类型注释**
可以使用特殊的行尾注释 `# type: ignore` 来禁用该行的类型检查. `pytype` 针对特定错误有一个禁用选项(类似lint):
```python
# pytype: disable=attribute-error
```

### **变量类型注解**
当一个内部变量难以推断其类型时,可以有以下方法来指示其类型:

 - 使用行尾注释 `# type:`:
	```python
    a = SomeUndecoratedFunction()  # type: Foo
	```
 - **带类型注解的复制** 如函数形参一样,在变量名和等号间加入冒号和类型:
	```python
	a: Foo = SomeUndecoratedFunction()
	```

### **泛型**
在注释时,尽量将泛型类型注释为类型参数.否则, 泛型参数将被视为是 [Any](https://www.python.org/dev/peps/pep-0484/#the-any-type) .
```python
# good
def get_names(employee_ids: List[int]) -> Dict[int, Any]:
	...

# bad
# These are both interpreted as get_names(employee_ids: List[Any]) -> Dict[Any, Any]
def get_names(employee_ids: list) -> Dict:
	...
```
若实在要用 Any 作为泛型类型,请显式的使用它。但在多数情况下, `TypeVar` 通常可能是更好的选择.
```python
def get_names(employee_ids: List[Any]) -> Dict[Any, Text]:
    """Returns a mapping from employee ID to employee name for given IDs."""

T = TypeVar('T')
def get_names(employee_ids: List[T]) -> Dict[T, Text]:
    """Returns a mapping from employee ID to employee name for given IDs."""
```

**特殊类型**
```python
"""Example"""
import asyncio
from typing import Callable, Any, Type, Tuple, Dict, Optional
from functools import partial

class BaseTask:
    """base Task"""

    def run(self) -> bool:
        """Run task"""
        raise NotImplementedError

    def stop(self) -> None:
        """Stop task"""
        raise NotImplementedError


class FileTask(BaseTask):
    """File task"""

    def run(self) -> bool:
        pass

    def stop(self) -> None:
        pass


class NetworkTask(BaseTask):
    """Network task"""

    def run(self) -> bool:
        pass

    def stop(self) -> None:
        pass


KwargsType = Dict[str, Any]
ArgsType = Tuple[Any]


async def run_in_executor(
        func: Callable[..., Any],
        args: Optional[ArgsType] = (),
        kwargs: Optional[KwargsType] = None
) -> Any:
    """Wrap a func in a threading executor """
    if kwargs:
        func = partial(func, **kwargs)
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(None, func, *args)


def task_runner(task_kls: Type[BaseTask]) -> None:
    """Task runner"""
    task = task_kls()
    asyncio.run(run_in_executor(task.run))
```
在  `run_in_executor`  方法中使用了  `typing.Callable`  、  `typing.Optional`  、  `typing.Any`  特殊类型。

在  `task_runner`  中使用  `typing.Type`  类型，表明  `task_kls`  参数是一个  `BaseTask`  类自身， 而不是它的对象，准确说是它的类对象。由于`FileTask`，`NetworkTask`，继承自`BaseTask`  类，因此这3个类的对象实例都可以传入作为`task_runner`的参数。



## TODO注释
`TODO`一般是为了暂时还未完成的代码作一个简单的提醒，可以是自己也可以是其他开发者。为临时代码使用``TODO``注释，它是一种短期解决方案。不算完美，但够好了。
`TODO`注释应该在所有开头处包含`TODO`字符串，紧跟着是用括号括起来的你的名字，email地址或其它标识符。然后是一个可选的冒号。接着必须有一行注释，解释要做什么。主要目的是为了有一个统一的`TODO`格式，这样添加注释的人就可以搜索到(并可以按需提供更多细节)。写了`TODO`注释并不保证写的人会亲自解决问题。当你写了一个`TODO`，请注上你的名字。
```python
# TODO(kl@gmail.com): Use a "*" here for string repetition.
# TODO(Zeke) Change this to use relations.
```
如果你的`TODO`是”将来做某事”的形式，那么请确保你包含了一个指定的日期(“2009年11月解决”)或者一个特定的事件(“等到所有的客户都可以处理XML请求就移除这些代码”)。

## 命名

 - 包名：`package_name`
 - 模块名：`module_name`
 - 类名：`ClassName`
 - 方法名：`method_name`
 - 类名：`ExceptionName`
 - 函数名: `function_name`
 - 全局常量名: `GLOBAL_CONSTANT_NAME`
 - 全局变量名: `global_var_name`
 - 实例名: `instance_var_name`
 - 函数参数名: `function_parameter_name`
 - 局部变量名: `local_var_name`

**函数名**、**变量名**和**文件名**应该是描述性的，尽量**避免缩写**，特别要避免使用非项目人员不清楚难以理解的缩写，不要通过删除单词中的字母来进行缩写。

**命名约定**

 - 使用单下划线`_`开头表示模块变量或者函数是`protected`的，即只应该模块内或者函数内使用（硬是访问的话也是可以的），不应该被公开访问，而且python解析时使用`from module import *`时该变量或函数不会被包含。
 - 用双下划线(`__`)开头的实例变量或方法表示类内私有。即只应该模块内或者函数内使用，不应该被公开访问。使用`module.var`与`class.var`会报错（`该属性不存在`），不能访问（可以使用_class__var访问， 具体可以查看相关文档）。
 - 将相关的类和顶级函数放在同一个模块里。不像Java，没必要限制一个类一个模块。
 - 对类名使用大写字母开头的单词(如CapWords，即Pascal风格)，但是模块名应该用小写加下划线的方式(如lower_with_under.py)。因为如果模块名碰巧和类名一致, 这会让人困扰。

## 代码风格检测

为了让开发风格达到统一，使用代码格式化工具检测。

使用  `isort`  将导包部分格式化为统一格式，使用  `pylint`  检测代码是否符合 `PEP-8` 规范，同时还能检测 一些不标准的的语法，并给出修改建议。

执行  `isort . --check-only --diff`  检测代码风格，并仅输出不符合规范的导包，执行  `isort`  会自动格式化代码。

运行  `pylint src tests`  检查  `src`  目录和  `tests`  目录下的 Python 代码。会输出不符合规范的内容，然后 根据建议修改即可。

Pycharm等IDE都存在一键自动格式化代码的功能。例如在Pycharm中使用`Ctrl+Alt+L`快捷键或者
使用鼠标选定不规范（~提示）的地方，使用`Alt+Shift+Enter`快捷键一键格式化。
