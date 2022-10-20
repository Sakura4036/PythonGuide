# 配置文件
配置是一个项目的核心驱动，可以在不更改源代码或减少源代码修改的情况下快速调整项目的运行。 使用中心配置驱动项目，能让项目的使用更加灵活，运维工作更轻松。

-   提高了代码的重用性，不再每次都去修改代码内部
-   这意味着其他不太懂你代码内部的人也可以使用你的项目，只用根据需求更改配置即可
-   有利于团队协作
-   有利于安全数据/秘密数据的管理

通过配置文件驱动项目的相关基础配置，如默认`data`和`logs`文件夹的所在路径和名称，Web项目相关配置，数据库连接相关配置，第三方库使用配置等。这些不同配置可以分别存放在不同的配置文件中，且文件类型可以多种多样，如`.py`，`.cfg`，`.ini`，`.yaml`，`.json`，`.txt`等。配置文件使用方法参考这个[博客](https://www.cnblogs.com/wanglvtao/p/11140025.html)。

## 静态配置 
参考：[source](https://pyloong.github.io/pythonic-project-guidelines/guidelines/advanced/configuration/)

在很多开源项目或者一些较小的项目中常见对配置文件的使用做法是：
配置文件写在一个或多个python文件中，比如上文的	`settings.py`。项目中哪个模块用到这个配置文件就直接通过`import settings`这种形式来在代码中使用配置。
```python
## Settings

# File config
SOURCE_FILE = '/tmp/foo.txt'

# Log config
LOG_LEVEL = 'DEBUG'
LOG_FORMAT = '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
```
```python
## main
import config
import logging

logging.basicConfig(
	level=settings.LOG_LEVEL,
	format=settings.LOG_FORMAT
)
```
这种做法不太好的方面:
 - 列表这让单元测试变得困难（因为模块内部依赖了外部配置）
 - 另一方面配置文件作为用户控制程序的接口，应当可以由用户自由指定该文件的路径。
 - 程序组件可复用性太差，因为这种贯穿所有模块的代码硬编码方式，使得大部分模块都依赖`conf.py`这个文件。

## 动态配置
 所以，更好的方式是:
 - 模块的配置都是可以灵活配置的，不受外部配置文件的影响
 - 程序的配置也是可以灵活控制的。
 
常见的配置管理工具有： **cofigureparser**（python内置），**Dynaconf**，**python-dotenv**， **Hydra**等。

**Dynaconf**（ [pypi](https://pypi.org/project/dynaconf/)， [官网](https://www.dynaconf.com/)）是一个灵活的中心配置管理工具，能够从不同的配置数据存储方式中读取配置，例如`.py`，`.redis`，`.ini`，`.json`文件，系统环境变量等等。

其具有如下特点：

-   加载多个配置源
-   配置分层
-   Django Flask 扩展
-   支持 Redis 和 Vault

在项目中新建配置文件  `settings.yml`
```yaml
## setting
foo: 1
bar: 2
```
新建配置模块 `config.py`
```python
## config.py
from dynaconf import Dynaconf

settings = Dynaconf(settings_files=['settings.yml'])
```
新建一个 `app.py` 文件，使用配置
```python
## app.py
from config import settings

print(settings.FOO)
print(settings.BAR)
```
然后运行  `python app.py`  可以看到已经能够自动获取  `settings.yml`  配置文件中的值。

增加本地配置文件  `settings.local.yml`
```yaml
## setting.local
foo: 10
bar: 20
```
再次运行  `python app.py` ，程序会自动获取  `settings.local.yml`  。

这是因为 `Dynaconf` 在初始化是传入了配置文件格式为  `settings.yml` ，在加载配置时，会同时查找 `settings.local.yml` 的配置文件。 并将两个配置文件的内容合并，如果存在相同变量， `settings.local.yml` 会覆盖 `settings.yml` 中的配置。


## 支持的文件类型
-   **.toml**  - Dynaconf 默认和推荐的格式.
-   **.yaml|.yml**  - 推荐用于Django项目.
-   **.json**  - 用于重用现有或导出的设置
-   **.ini**  - 用于重用旧设置
-   **.py**  -  **不推荐**  但支持向后兼容
-   **.env**  - 用于自动加载环境变量。
