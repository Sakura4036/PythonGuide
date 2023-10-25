# python 环境移植

## 在线版

### conda 版本

#### 项目环境安装

1.  安装 conda 或者 miniconda：

    - 添加到系统 PATH， 则命令行中可直接使用 conda 命令

    - 不添加到系统 PATH，则需要到安装目录下的 `Scripts` 下使用 activate 启动 conda

2.  创建对应的虚拟环境， 例如：`conda activate -n py39 python=3.9`

3.  安装项目所需环境：&#x20;

    `pip install -r requirements.txt` 或者直接一条一条安装

    - torch，github 等特殊包使用对应的语句进行安装:

      - `pip install torch==1.10.0 torchvision==0.11.0 torchaudio==0.11.0 -f https://download.pytorch.org/whl/torch_stable.html `

      - `pip install torch-scatter==2.0.9 torch-sparse==0.6.13 torch-cluster==1.6.0 torch-spline-conv==1.2.1 torch-geometric==2.0.4 -f https://data.pyg.org/whl/torch-1.10.0+cpu.html`

4.  进入项目，运行: `python xxx.py`

5.  若不存在`requirements.txt` ，导出`requirements.txt` ：

    `pip freeze > requirements.txt`

#### 项目环境移植

安装步骤同`项目环境安装`&#x20;

### 2. 直接安装/venv 安装

> **conda 环境也可以使用 python3 安装 venv**

#### 项目环境安装

1.  安装指定版本 python，例如 python 3.9

    - [python 3.9.13](https://www.python.org/downloads/release/python-3913/)最后一个提供二进制安装程序的 python 版本

    - [python 3.9.14](https://www.python.org/downloads/)及以上版本只提供源码安装： `setup.py install`

2.  使用 python 自带 venv 创建环境（也可不使用）：

    `python -m venv py39`

3.  激活 venv 环境：

    `.\py39\Scripts\activate`

4.  安装环境，`pip install -r requirements.txt` 或者直接一条一条安装

5.  进入项目，运行: `python xxx.py`

6.  若不存在`requirements.txt` ，导出`requirements.txt` ：

    `pip freeze > requirements.txt`

#### 项目环境移植

安装步骤同**项目环境安装**

## 离线移植

### 离线 python 安装

1.  python 二进程程序下载

2.  python 源码安装

> python 下载官网：<https://www.python.org/downloads/>

### 环境移植

### &#x20;venv 打包移植

#### 直接环境打包移植

1.  直接将原主机下安装的 venv 文件夹进行打包，例如 venv 环境 py39 打包成 py39.zip，发送至目标主机，项目文件夹下或者任意文件夹下解压

2.  **修改 venv 启动脚本**：

    `venv`的启动文件`activate、activate.bat、activate.sh`里，`python SDK`的`PATH`设置的是原`python SDK`的环境，项目移动后，需要更新这几个文件里的`PATH`变量

    - windows：需要修改`venv\Scripts`下的`Activate`和`activate.bat`两个文件中的`$env:VIRTUAL_ENV`,改成当前工程的`venv`路径

    - 需要修改`venv\bin`下的`Activate`文件中的`$env:VIRTUAL_ENV`,改成当前工程的`venv`路径

3.  **上述文件修改之后工程已经可以正常激活，但是如果安装了 pip 后的并不能用**

    - **修改 pip 文件中的路径（一般在首行）为当前工程 venv 环境下的 python 路径**

4.  修改 venv 使用的基础 python 路径：**进入 venv 文件夹，修改 pyvenv.cfg 文件中的 home 参数，指定该 venv 使用的 python**

    > venv 文档：<https://docs.python.org/zh-cn/3/library/venv.html>

    ![](python环境移植_md_files/af6097b0-724f-11ee-9291-81a2f7769dad_20231024172826.jpeg?v=1&type=image&token=V1%3Ab0DxX47pmUj-ZT3HjOkbh8xRAlN3-UxtICVpmF3gOnI)

5.  运行项目进行测试

#### pip packages 安装包移植

1.  首先激活虚环境，导出环境：`pip freeze > requirements.txt`

2.  使用 pip 命令生成批量离线安装包（whl 文件）：`pip wheel --wheel-dir=./tmp/packages -r requirements.txt` ，检查/tmp/packages 中是否包含了项目所需的所有.whl。

    **如果运行报错，有部分包无法打包成 wheel：**

    1.  请先将这些错误包移除`requirements.txt` 然后重新执行`pip wheel --wheel-dir=./tmp/packages -r requirements.txt`

    2.  torch，torch_geometric 等相关包无法从 pip 下载可以使用以下命令：

        1.  写入 torch 等指定版本至**requirements_torch.txt**：

            > torch\=\=1.11.0+cu113\
            > torchaudio\=\=0.11.0+cu115\
            > torchvision\=\=0.12.0+cu113

        2.  执行`pip wheel --wheel-dir=./tmp/packages -r requirements_torch.txt -f https://download.pytorch.org/whl/torch_stable.html`

        3.  写入 torch_geometric 等指定版本至**requirements_tgl.txt**：

            > torch-cluster\=\=1.6.0\
            > torch-scatter\=\=2.0.9\
            > torch-sparse\=\=0.6.13\
            > torch-spline-conv\=\=1.2.1\
            > torch_geometric\=\=2.4.0

        4.  执行`pip wheel --wheel-dir=./tmp/packages -r requirements_tgl.txt -f https://download.pytorch.org/whl/torch_stable.html`

    3.  其他错误包参考下面的**pip wheel 失败解决办法，生成 whl 之后复制到**`./tmp/packages`

    4.  最后将错误的包及 torch，torch_geometric 等加入到`requirements.txt`中，即与原环境 freeze 得到的`requirements.txt` 内容一致

3.  打包至目标主机，创建虚拟环境（前提以及装好 python）并激活：`python -m venv venv_name` ，`.\venv\Scripts\activate`

4.  安装打包的依赖模块：`pip3 install --no-index --find-links=tmp/packages -r requirements.txt`

5.  安装完毕，检查：`pip3 freeze` 或者运行项目检查

> **Python 离线项目迁移部署：**<https://zhuanlan.zhihu.com/p/114290069>

#### pip wheel 失败解决办法：

> **Python package 只有压缩格式如何转成 wheel 格式：**<https://blog.csdn.net/linkeeee/article/details/130618985>

1.  在[pypi](https://pypi.org/)使用 package 名称进行搜索，download 界面选择指定版本下载压缩文件

2.  解压，cmd 到当前解压文件夹（setup.py 所在目录）

3.  检查 package 的 setup.py 文件中的配置是否正确：`python setup.py check`

4.  生成 wheel：`python setup.py sdist bdist_wheel || true`

5.  在生成的 dist 文件夹中查看生成的 wheel 文件

### pyinstaller 打包移植

1.  下载 pyinstaller

2.  项目打包配置

3.  运行打包命令

4.  复杂至目标主机

5.  解压运行
