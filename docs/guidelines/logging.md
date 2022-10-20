# 日志
参考：[source](https://pythonguidecn.readthedocs.io/zh/latest/writing/logging.html)


日志是对软件执行时所发生事件的一种追踪方式。软件开发人员对他们的代码添加日志调用，借此来指示某事件的发生。 一个事件通过一些包含变量数据的描述信息来描述（比如：每个事件发生时的数据都是不同的）。开发者还会区分事件的重要性， 重要性也被称为 **等级** 或 **严重性**。有一个好的日志实践，能让开发调试流程更顺畅，出现问题能更快速精准定位。[日志](https://docs.python.org/2/library/logging.html#module-logging)  模块自2.3版本开始便是Python标准库的一部分。它被简洁的描述在  [**PEP 282**](https://www.python.org/dev/peps/pep-0282)。 

日志的两个目的：

-   **诊断日志**  记录与应用程序操作相关的日志。例如，用户遇到的报错信息， 可通过搜索诊断日志获得上下文信息。
-   **审计日志**  为商业分析而记录的日志。从审计日志中，可提取用户的交易信息， 并结合其他用户资料构成用户报告或者用来优化商业目标。

## 打印（Print）
当需要在命令行应用中显示帮助文档时，  `print`  是一个相对于日志更好的选择。 而在其他时候，日志总能优于  `print`  ，理由如下：

-   日志事件产生的  [日志记录](https://docs.python.org/library/logging.html#logrecord-attributes)  ，包含清晰可用的诊断信息，如文件名称、路径、函数名和行号等。
-   包含日志模块的应用，默认可通过根记录器对应用的日志流进行访问，除非您将日志过滤了。
-   可通过  [`logging.Logger.setLevel()`](https://docs.python.org/3/library/logging.html#logging.Logger.setLevel)  方法有选择地记录日志， 或可通过设置  `logging.Logger.disabled`  属性为  `True`  来禁用。

## 配置日志
-   使用**INI**格式文件：
    
    -   **优点**: 使用  [`logging.config.listen()`](https://docs.python.org/3/library/logging.config.html#logging.config.listen "(在 Python v3.7)")  函数监听socket，可在运行过程中更新配置
    -   **缺点**: 通过源码控制日志配置较少（  _例如_  子类化定制的过滤器或记录器）。
-   使用**字典**或**JSON**格式文件：
    
    -   **优点**: 除了可在运行时动态更新，在Python 2.6之后，还可通过  [`json`](https://docs.python.org/3/library/json.html#module-json")  模块从其它文件中导入配置。
    -   **缺点**: 很难通过源码控制日志配置。
    
-   使用**源码**：
    -   **优点**: 对配置绝对的控制。
    -   **缺点**: 对配置的更改需要对源码进行修改。

### 通过INI文件进行配置的例子

我们假设文件名为  `logging_config.ini`  。关于文件格式的更多细节，请参见  [日志指南](http://docs.python.org/howto/logging.html)  中的  [日志配置](https://docs.python.org/howto/logging.html#configuring-logging)  部分。
```ini
[loggers]
keys=root

[handlers]
keys=stream_handler

[formatters]
keys=formatter

[logger_root]
level=DEBUG
handlers=stream_handler

[handler_stream_handler]
class=StreamHandler
level=DEBUG
formatter=formatter
args=(sys.stderr,)

[formatter_formatter]
format=%(asctime)s %(name)-12s %(levelname)-8s %(message)s
```
然后在源码中调用 `logging.config.fileConfig()` 方法：
```python
import logging
from logging.config import fileConfig

fileConfig('logging_config.ini')
logger = logging.getLogger()
logger.debug('often makes a very good meal of %s', 'visiting tourists')
```

### 通过字典进行配置的例子

Python 2.7中，您可以使用字典实现详细配置。[**PEP 391**](https://www.python.org/dev/peps/pep-0391)  包含了一系列字典配置的强制和 非强制的元素。

```python
import logging
from logging.config import dictConfig

logging_config = dict(
    version = 1,
    formatters = {
        'f': {'format':
              '%(asctime)s  %(name)-12s  %(levelname)-8s  %(message)s'}
        },
    handlers = {
        'h': {'class': 'logging.StreamHandler',
              'formatter': 'f',
              'level': logging.DEBUG}
        },
    root = {
        'handlers': ['h'],
        'level': logging.DEBUG,
        },
)

dictConfig(logging_config)

logger = logging.getLogger()
logger.debug('often makes a very good meal of %s', 'visiting tourists')
```

### 通过源码直接配置的例子
```python
import logging

logger = logging.getLogger()
handler = logging.StreamHandler()
formatter = logging.Formatter(
        '%(asctime)s  %(name)-12s  %(levelname)-8s  %(message)s')
handler.setFormatter(formatter)
logger.addHandler(handler)
logger.setLevel(logging.DEBUG)

logger.debug('often makes a very good meal of %s', 'visiting tourists')
```


### logger文件示例
```python
# logger.py
# 创建Logger对象实例logger，然后其他模块只需要 from logger import logger
# 通过logger.info("")方法调用


def create_logger(log_file, level=logging.DEBUG):  
    # logging.basicConfig函数对日志的输出格式及方式做相关配置  
  logging.basicConfig(level=level,  
  format='%(asctime)s - %(filename)s[line:%(lineno)d] - %(levelname)s: %(message)s')  
  log = logging.getLogger()  
  log.setLevel(logging.INFO)  
  
  if not os.path.exists(os.path.dirname(log_file)):  
        os.makedirs(os.path.dirname(log_file))  
  # 创建一个handler，用于写入日志文件  
  fh = logging.FileHandler(log_file, mode='w')  
  fh.setLevel(level)  
  # 定义handler的输出格式  
  formatter = logging.Formatter("%(asctime)s - %(filename)s[line:%(lineno)d] - %(levelname)s: %(message)s")  
  fh.setFormatter(formatter)  
  
  # 将log添加到handler里面  
  log.addHandler(fh)  
  return log
  
log_file_dir = 'logs'
if not os.path.exists(log_file_dir):  
    os.makedirs(log_file_dir)  
rq = time.strftime('%Y-%m-%d-%H-%M', time.localtime(time.time()))  
log_file = os.path.join(log_file_dir, rq + '.log')
logger = create_logger(log_file)

 
```
