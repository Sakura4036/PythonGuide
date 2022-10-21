# 文档管理

项目文档用来说明和记录项目的信息，有助于开发人员、管理人员、使用者的交流和沟通。在 Python 项目中 一般通过 [Mkdocs](https://www.mkdocs.org/) 和 [sphinx](https://www.sphinx-doc.org/en/master/) 来 构建项目文档。两者都支持 markdown 标记的文件，但后者也支持 [reStructuredText](https://docutils.sourceforge.io/rst.html) 标记文件。

##  Readme
**Readme**与文档管理中的文档是不同的，各自有各自的作用。

 - [Readme](../project_structure/#readme)：这应该是每个项目都应该有的一个文件，目的是能简要描述该项目的信息，让读者快速了解这个项目。Readme负责简单介绍项目背景、软件功能、简单的使用说明、常见问题等等信息。
 - 文档（docs）：文档负责记录整个项目的具体信息，包括项目结构、模块功能实现、开发日志、版本控制等等。


## MkDocs
MkDocs是一个快速、简单、华丽的静态网站生成器，适用于构建项目文档。文档源文件以Markdown编写，并使用一个YAML文件来进行配置。 MkDocs生成完全静态的HTML网站，你可以将其部署到GitHub pages、Amzzon S3或你自己选择的其它任意地方。
MkDocs有一堆很好看的主题。 官方内置了两个主题： _mkdocs_ 和 _readthedocs_， 也可以从[MkDocs wiki](https://github.com/mkdocs/mkdocs/wiki/MkDocs-Themes)中选择第三方主题， 或者[自定义主题](https://mkdocs.zimoapps.com/user-guide/custom-themes/)。你还可以使用第三方库[mkdocs-material](https://squidfunk.github.io/mkdocs-material/)来使用更多更好看的主题库以及更强大的插件。

- [MkDocs官网](https://www.mkdocs.org/)
- [MkDocs官方中文文档](https://mkdocs.zimoapps.com/)
- [MkDocs-Material](https://squidfunk.github.io/mkdocs-material/)

特点：

-   YAML 单文件配置
-   生成静态站点
-   支持 markdown
-   支持自定义主题
-   支持 markdown 扩展标记
-   支持插件

**注：** 本网站也是使用通过编写MarkDown文件，然后使用MkDocs与mkdocs-material生成的静态网站。

## Sphinx
Sphinx 是一个  _文档生成器_  ，您也可以把它看成一种工具，它可以将一组纯文本源文件转换成各种输出格式，并且自动生成交叉引用、索引等。也就是说，如果您的目录包含一堆  [reStructuredText](https://www.sphinx-doc.org/zh_CN/master/usage/restructuredtext/index.html)  或  [Markdown](https://www.sphinx-doc.org/zh_CN/master/usage/markdown.html)  文档，那么 Sphinx 就能生成一系列HTML文件，PDF文件（通过LaTeX），手册页等。

Sphinx 专注于文档，尤其是 handwritten documentation ，然而，Sphinx 也可以用来生成博客、主页甚至书籍。Sphinx 的大部分功能来自于 reStructuredText ，它是一种纯文本标记格式，有着丰富的功能和  [显著的扩展能力](https://www.sphinx-doc.org/zh_CN/master/development/index.html)  。例如[Google Python风格指南中文版](https://zh-google-styleguide.readthedocs.io/en/latest/google-python-styleguide/contents/)就是使用 _Sphinx_ 与 _reStructuredText_ 文档生成的网站。

- [Sphinx官网](https://www.sphinx-doc.org/en/master/)
- [Sphinx官方中文文档](https://www.sphinx-doc.org/zh_CN/master/usage/index.html)

特点：

-   单个 Python 文件配置
-   生成 HTML 、 ePub 等多种格式
-   支持 markdown 和 reStructuredText
-   支持自定义主题
-   支持扩展




