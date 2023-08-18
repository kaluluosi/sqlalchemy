.. _overview_toplevel:
.. _overview:

========
概述
========

SQLAlchemy SQL Toolkit和ORM(Object Relational Mapper)是一个用于处理数据库和Python的完整工具集。它有几个独立领域的功能，这些功能可以单独使用或结合使用。其主要组成部分如下图所示，组件之间的依赖关系组织成层：

.. image:: sqla_arch_small.png

上图中，SQLAlchemy最显著的前端部分是**ORM(Object Relational Mapper)**和**Core**。

Core包含了SQLAlchemy的主要SQL和数据库集成与描述服务，其中最突出的部分是**SQL Expression Language**。

SQL Expression Language本身是一种工具包，独立于ORM包，提供了通过可组合的对象构建SQL表达式的系统，然后可以对特定事务范围内的目标数据库执行这些表达式，返回结果集。插入、更新和删除（即:DML）是通过传递表示这些语句的SQL表达式对象以及表示要与每个语句一起使用的参数的字典来实现的。

ORM基于Core来提供一种处理与数据库架构映射的域对象模型的方式。当使用ORM时，SQL语句的构建方式与使用Core时基本相同，但是DML（这里指业务对象在数据库中的持久化）的任务是通过称为**unit of work**的模式自动化，在该模式中，将可变对象的状态更改转换为INSERT、UPDATE和DELETE构造。然后根据这些对象的术语调用。SELECT语句也通过ORM特定的自动化和面向对象的查询功能进行增强。

与使用Core和SQL Expression语言工作提供了数据库的基于架构的视图以及基于不可变性的编程范例相比，ORM在此基础上建立了一个基于领域的数据库视图，其编程范例更为明确地面向对象并且依赖于可变性。因为关系数据库本身就是一项可变的服务，所以与Core/ SQL Expression语言的区别在于Core/SQL Expression语言是命令定向的而ORM是状态定向的。


.. _doc_overview:

文档概述
======================

文档分为四个部分:

* :ref:`unified_tutorial` - 此1.4/2.0系列的全新教程综合介绍了整个库，从Core的描述开始，逐步向ORM特定的概念工作。新用户和来自SQLAlchemy 1.x系列的用户应从这里开始。

* :ref:`orm_toplevel` - 在这个部分，展示了ORM的参考文档。

* :ref:`core_toplevel` - 在这里，展示了Core中的所有其他内容的参考文档。SQLAlchemy引擎、连接和池服务也在此处得到介绍。

* :ref:`dialect_toplevel` - 提供了所有dialect实现的参考文档，包括DBAPI专用词。


代码示例
=============

包含有关ORM的可操作代码示例已包括在SQLAlchemy发行版中。所有已包括示例应用的说明在:ref:`examples_toplevel`中。

在wiki中还有大量有关Core SQLAlchemy结构以及ORM的示例。有关详细信息，请参见`Theatrum Chemicum<https://www.sqlalchemy.org/trac/wiki/UsageRecipes>`_。


.. _installation:

安装指南
==================

支持平台
-------------------

SQLAlchemy支持以下平台:

* cPython 3.7 及更高版本
* 适用于`PyPy <http://pypy.org/>`_的 Python 3 兼容版本

.. versionchanged:: 2.0
   SQLAlchemy现在针对Python 3.7及以上版本进行目标定位。

AsyncIO支持
----------------

SQLAlchemy的``asyncio``支持取决于`greenlet <https://pypi.org/project/greenlet/>`_项目。此依赖项将在常见机器平台上默认安装，但不会支持所有架构，并且也可能不会在不常见的架构上默认安装。请参见：ref:`asyncio_install`节，以了解有关确保asyncio支持正常的其他详细信息。

支持的安装方法
-------------------------------

SQLAlchemy的安装是通过基于`setuptools <https://pypi.org/project/setuptools/>`_的标准Python方法进行的，可以直接引用``setup.py``或使用`pip <https://pypi.org/project/pip/>`_或其他支持setuptools的方法进行。

通过pip进行安装
---------------

如果``pip``可用，可以使用以下一步下载SQLAlchemy分配并进行安装：

.. sourcecode:: text

    pip install SQLAlchemy

此命令将从`Python Cheese Shop <https://pypi.org/project/SQLAlchemy>`_下载SQLAlchemy的最新“released”版本，并将其安装到系统中。对于大多数常见平台，将下载提供本地Cython / C扩展预构建的Python Wheel文件。

要安装最新的“prerelease”版本，例如“2.0.0b1”，需要使用``--pre``标志：

.. sourcecode:: text

    pip install --pre SQLAlchemy

如果最近的版本是预发行版，则将安装它，而不是最新发布的版本。

从源代码分发手动安装
-------------------------------------------------

如果不是从pip安装，则可以使用``setup.py``脚本安装源分发：

.. sourcecode:: text

    python setup.py install

源代码安装与平台无关，可以在任何平台上安装，无论是否安装了Cython / C构建工具。正如下一节:ref: `c_extensions`所述，``setup.py``将尝试使用Cython / C进行构建，如果不可能，则会回退到纯Python安装。

.. _c_extensions:

构建Cython扩展
----------------------------------

SQLAlchemy包含Cython_扩展，这些扩展提供了各种领域内的额外速度提升，目前重点关注Core结果集的速度。

.. versionchanged:: 2.0 SQLAlchemy C扩展已使用Cython进行重写。

如果检测到一个合适的平台，``setup.py``将自动构建扩展，假设Cython包已安装。完整的手动构建如下所示：

.. sourcecode:: text

    # 进入SQLAlchemy源分销商
    cd path/to/sqlalchemy

    # 安装Cython
    pip install cython

    # 可选地提前安装Cython扩展
    python setup.py build_ext

    # 运行安装
    python setup.py install

也可以使用:pep:`517`技术，例如使用库构建源/ wheel分销商：

.. sourcecode:: text

    # 进入SQLAlchemy源分销商
    cd path/to/sqlalchemy

    # 安装build
    pip install build

    ＃构建源码/轮Dists
    python -m build

如果Cython扩展的构建因未安装Cython、缺少编译器或其他问题而失败，则设置过程将输出警告消息，并在完成后在Cython扩展上重新运行构建并报告最终状态。


要完全运行构建/安装而无需尝试编译Cython扩展，可以指定``DISABLE_SQLALCHEMY_CEXT``环境变量。用例是特殊测试情况，或在通常的“重建”机制无法克服的兼容性/构建问题的罕见情况下：

.. sourcecode:: text

  export DISABLE_SQLALCHEMY_CEXT=1; python setup.py install


.. _Cython: https://cython.org/

.. _build: https://pypi.org/project/build/


安装数据库API
----------------------------------

SQLAlchemy旨在与专为特定数据库构建的:term:`DBAPI`实现一起运行，并包含对最受欢迎的数据库的支持。在:doc:`/dialects/index`中的各个数据库部分枚举了每个数据库的可用DBAPI，包括外部链接。

检查已安装的SQLAlchemy版本
------------------------------------------

此文档涵盖的SQLAlchemy版本为2.0。如果您正在使用已安装SQLAlchemy的系统，请像这样从Python提示符中检查版本：

     >>> import sqlalchemy
     >>> sqlalchemy.__version__  # doctest: +SKIP
     2.0.0

下一步
----------

安装SQLAlchemy后，无论是新用户还是老用户，都可以 **下一步** :ref:`继续SQLAlchemy教程 <unified_tutorial>`。

.. _migration:

1.x 到 2.0 迁移
=====================

SQLAlchemy 2.0发布的新API说明在 :doc:`changelog/migration_20`中。